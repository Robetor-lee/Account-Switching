---
name: hermes-dual-gateway
description: "Use when the user asks to configure, set up, or debug multiple Hermes profiles running simultaneously with independent gateways on different ports and platforms. Covers dual-gateway architecture, profile isolation, startup scripts, scheduled tasks, cron job separation, and common pitfalls like --replace without --profile."
version: 2.0.0
author: Robetor-lee
license: MIT
platforms: [windows, linux, macos]
metadata:
  hermes:
    tags: [gateway, profiles, dual-profile, multi-account, devops]
    related_skills: [hermes-profile-debugging, hermes-gateway-ops]
---

# Hermes Dual Gateway Setup

Configure multiple Hermes profiles to run simultaneously as independent gateway processes, each with its own communication platform, config, env, cron jobs, skills, and memory. Zero coupling between profiles.

## Overview

Each Hermes profile is a complete independent runtime identified by a unique `HERMES_HOME` directory. When running multiple profiles simultaneously, each needs its own gateway process on a unique port, bound to exactly one communication platform. The architecture uses environment variable isolation (not CLI flags) to keep profiles from interfering with each other.

Typical use cases: personal vs work identity separation, different security postures per use case (interactive with approvals vs autonomous with YOLO), multi-platform distribution (wecom for team, weixin for self), isolated cron job namespaces.

## When to Use

Load this skill when:
- User asks to run multiple Hermes profiles or gateways at the same time
- User wants to separate personal and work Hermes identities
- User needs different security/approval settings for different use cases
- User asks about profile isolation, `HERMES_HOME`, or multi-port gateway setups
- User asks to add a second (or third) gateway alongside an existing one
- User hits "Token already in use" errors and needs to understand why
- User is setting up cron jobs across multiple profiles and needs delivery isolation

## Architecture

```
Gateway Process A                  Gateway Process B
  port: 8641                         port: 8642
  HERMES_HOME=~/.hermes              HERMES_HOME=~/.hermes/profiles/<name>
  platform: wecom                    platform: weixin
       │                                    │
  ┌────┴───────────┐              ┌────────┴───────────┐
  │ config.yaml     │              │ config.yaml         │
  │ .env            │              │ .env  (independent) │
  │ skills/         │              │ skills/             │
  │ memories/       │              │ memories/           │
  │ cron/           │              │ cron/               │
  │ state.db        │              │ state.db            │
  │ logs/           │              │ logs/               │
  └─────────────────┘              └─────────────────────┘
```

Core invariants:
- One profile = one gateway = one platform = one port
- Profile `.env` does NOT inherit from root `.env` — every env var must be defined locally
- Platform assignment is mutually exclusive: if profile A uses wecom, profile B MUST set `wecom.enabled: false`
- Cron DBs are independent per profile (separate `cron/` directories)

## Profile Configuration

### Step 1: Plan the profile layout

Before creating files, confirm with the user:

1. What to name the new profile (lowercase, hyphens, e.g. `work`, `personal`, `scraper`)
2. Which port to use (8641 is default; use 8642, 8643, etc. for additional profiles)
3. Which communication platform (wecom, weixin, telegram, discord, etc.)
4. What security posture (interactive with approvals, or autonomous YOLO)
5. Is this profile for cron-only, interactive-only, or both

### Step 2: Create profile directory structure

```bash
PROFILE="<name>"
mkdir -p ~/.hermes/profiles/$PROFILE/{skills,memories,cron,logs,gateway-service,desktop,cache}
```

### Step 3: Write the profile config.yaml

Create `~/.hermes/profiles/<name>/config.yaml`. The critical sections:

```yaml
model:
  base_url: https://api.deepseek.com/v1
  default: deepseek-v4-pro
  provider: deepseek

platforms:
  api_server:
    enabled: true
    key: ${API_SERVER_KEY}
    extra:
      port: <unique-port>     # e.g. 8642
      host: 127.0.0.1
  <chosen-platform>:
    enabled: true
    # ... platform-specific auth keys as ${ENV_VAR} references
  <all-other-platforms>:
    enabled: false            # EXPLICITLY disable every platform this profile does NOT own

# For autonomous profiles:
approvals:
  mode: false
security:
  tirith_enabled: false
  allow_private_urls: true
hooks_auto_accept: true
delegation:
  subagent_auto_approve: true
```

**CRITICAL**: Every platform this profile does NOT use must have `enabled: false`. Never leave a platform unconfigured — it will try to start and fail or conflict.

### Step 4: Write the profile .env

Create `~/.hermes/profiles/<name>/.env`. Copy only the vars this profile needs. Do NOT assume root `.env` is inherited — it is NOT. Every `${VAR}` in config.yaml must resolve from this file.

At minimum:
```bash
DEEPSEEK_API_KEY=sk-xxx
API_SERVER_KEY=your-secret
# Platform-specific keys...
```

### Step 5: Also update the root profile's config

The root profile (`~/.hermes/config.yaml`) must explicitly disable the platform the new profile will own:

```yaml
platforms:
  <platform-new-profile-uses>:
    enabled: false    # ADD this line to the root config
```

## Gateway Startup Scripts

### Windows (Scheduled Tasks)

Create `~/.hermes/profiles/<name>/gateway-service/Hermes_Gateway_<name>.cmd`:

```batch
@echo off
cd /d <path-to-hermes-agent>
set "HERMES_HOME=%USERPROFILE%\.hermes\profiles\<name>"
set "PYTHONIOENCODING=utf-8"
set "HERMES_GATEWAY_DETACHED=1"
set "VIRTUAL_ENV=<path-to-venv>"
<path-to-venv>\Scripts\pythonw.exe -m hermes_cli.main --profile <name> gateway run
exit /b 0
```

Register as a Scheduled Task for auto-start on login:

```bash
schtasks //create //tn Hermes_Gateway_<name> \
  //tr "%USERPROFILE%\.hermes\profiles\<name>\gateway-service\Hermes_Gateway_<name>.cmd" \
  //sc ONSTART //delay 0001:00 //f
```

On MSYS/git-bash: use `//` for schtasks flags to avoid path translation.

### Linux (systemd)

Create `/etc/systemd/system/hermes-gateway-<name>.service`:

```ini
[Unit]
Description=Hermes Agent Gateway - <name> profile
After=network.target

[Service]
Type=simple
User=<user>
Environment=HERMES_HOME=/home/<user>/.hermes/profiles/<name>
WorkingDirectory=<path-to-hermes-agent>
ExecStart=<path-to-venv>/bin/python -m hermes_cli.main --profile <name> gateway run
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now hermes-gateway-<name>
```

## Verification

After starting both gateways, verify:

```bash
# Check both ports are listening
netstat -ano | grep -E "8641|8642"     # Windows
ss -tlnp | grep -E "8641|8642"         # Linux

# Check gateway logs for errors
tail -50 ~/.hermes/logs/gateway.log
tail -50 ~/.hermes/profiles/<name>/logs/gateway.log

# Verify platform-specific warnings are absent
grep -i "warning\|error" ~/.hermes/profiles/<name>/logs/gateway.log | grep -v "Token already in use"
```

## Cron Jobs Across Profiles

When creating cron jobs from a profile session, use the `profile` field to ensure the job runs under the correct profile's config and delivers to the correct platform:

```python
cronjob(action='create', profile='<name>', schedule='0 9 * * *', prompt='...', deliver='<target>')
```

If creating a cron job without `profile`, it inherits the current session's profile. Always double-check by listing jobs after creation.

Different profiles' cron DBs live at:
- Root: `~/.hermes/cron/`
- Named profile: `~/.hermes/profiles/<name>/cron/`

## Common Pitfalls

1. **`--replace` without `--profile` kills ALL gateways.** The `--replace` flag scans for any running gateway process from the same install regardless of profile. Always use `--profile <name>` explicitly. Never put `--replace` in a startup script.

2. **Profile .env inherits nothing from root.** If a config.yaml references `${WEIXIN_TOKEN}`, that var must exist in THAT profile's `.env` file or system environment — the root `.env` is not consulted. This is the number one cause of silent failures after adding a new profile.

3. **Forgetting to disable platforms in the other profile.** If profile A enables wecom and profile B doesn't explicitly set `wecom.enabled: false`, profile B will try to start a wecom adapter and either fail or cause token conflicts.

4. **Same port on two profiles.** Each profile's `api_server.extra.port` MUST be unique. Two gateways on the same port = second one fails to bind.

5. **Mixing platforms in one profile.** Enabling both wecom and weixin in the same profile causes token conflicts and unpredictable message routing. One profile = one platform = one identity.

6. **Editing the wrong config file.** When troubleshooting, always confirm which `HERMES_HOME` is active. The config at `~/.hermes/config.yaml` and `~/.hermes/profiles/<name>/config.yaml` are different files.

7. **Assuming `skill_manage(action='create')` creates skills for the new profile.** It creates them in the CURRENT session's profile. Use `cross_profile=True` or switch profiles first.

## Verification Checklist

After completing a dual-gateway setup, verify:
- [ ] Both ports are LISTENING (`netstat` / `ss`)
- [ ] Each profile's config explicitly disables the other's platform
- [ ] Each profile's `.env` contains all `${VAR}` references used in its config
- [ ] Startup scripts set `HERMES_HOME` to the correct profile directory
- [ ] Startup scripts use `--profile <name>` (not `--replace` alone)
- [ ] Scheduled tasks / systemd services are enabled for auto-start
- [ ] Gateway logs show no unexpected errors (ignore "Token already in use")
- [ ] Cron jobs are created with the correct `profile` parameter and `deliver` target
- [ ] A test message through each platform reaches the correct gateway
