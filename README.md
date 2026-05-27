# Hermes Dual Gateway — 多 Profile 网关配置 Skill

一键安装，让 Hermes 学会如何配置和管理多账户、多网关并行运行。

## 安装方式（三选一）

```bash
# 方式 1：添加为 skill tap（推荐 — 后续可装此仓库全部 skill）
hermes skills tap add Robetor-lee/Account-Switching

# 方式 2：直接按路径安装单个 skill
hermes skills install Robetor-lee/Account-Switching/devops/hermes-dual-gateway

# 方式 3：从原始 URL 直接安装
hermes skills install https://raw.githubusercontent.com/Robetor-lee/Account-Switching/main/devops/hermes-dual-gateway/SKILL.md
```

安装后，在 Hermes 对话中说"帮我设置双网关"或"我要加一个 work profile"，Hermes 会自动加载此 skill 并按标准流程操作。

## 功能

告诉 Hermes 如何完成以下任务：

- 创建独立 profile（独立 config、env、skills、memories、cron、logs）
- 配置多 gateway 在不同端口并行运行
- 分配通讯平台（wecom / weixin / telegram / discord 等）互不冲突
- 注册开机自启（Windows Scheduled Task / Linux systemd）
- 隔离 cron 定时任务的投递渠道
- 避免 `--replace` 误杀、`.env` 继承缺失等常见陷阱

## 仓库结构

```
devops/
└── hermes-dual-gateway/
    └── SKILL.md        ← Hermes skill 规范格式
```

符合 [Hermes Skill Tap](https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills) 标准目录布局 `<category>/<name>/SKILL.md`。

## 协议

MIT
