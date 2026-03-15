# openclaw-config-guide

当用户想系统了解 OpenClaw 的配置结构、目录布局、各配置文件用途、systemd 部署方式、gateway / tailscale / devices / workspace / memory 等配置入口，或想把这些内容整理成操作指南、排障手册、迁移说明时，使用此 skill。

## 适用场景

当用户提出以下类型的问题时启用：
- “OpenClaw 的目录结构是什么样？”
- “openclaw 支持哪些配置？分别在哪个文件/目录？”
- “帮我梳理 openclaw.json、devices、systemd、workspace 的关系”
- “把当前机器上的 OpenClaw 配置整理成文档/指南/笔记”
- “反推我当时是怎么把 OpenClaw 配起来的”
- “给我一份 OpenClaw 运维备忘单 / 配置说明 / 部署笔记”

## 目标

输出一份面向操作者的、可直接使用的 OpenClaw 配置说明，至少覆盖：
- 主配置文件与目录位置
- systemd 托管文件位置与差异
- devices / bootstrap / paired / pending 的作用
- workspace 下 AGENTS.md / SOUL.md / USER.md / MEMORY.md / HEARTBEAT.md 等文件用途
- gateway / models / plugins / extensions / tailscale / session 等常见配置主题
- 修改后如何 reload / restart / verify
- 哪些文件适合手改，哪些更适合只读检查

## 操作步骤

1. 优先读取本机实际存在的配置文件，而不是只讲通用概念。
2. 至少检查这些路径（存在则读）：
   - `~/.openclaw/openclaw.json`
   - `~/.openclaw/.env`
   - `~/.openclaw/devices/bootstrap.json`
   - `~/.openclaw/devices/paired.json`
   - `~/.openclaw/devices/pending.json`
   - `~/.config/systemd/user/openclaw-gateway.service`
   - `/etc/systemd/system/openclaw-gateway.service`
   - `~/.openclaw/workspace/AGENTS.md`
   - `~/.openclaw/workspace/SOUL.md`
   - `~/.openclaw/workspace/USER.md`
   - `~/.openclaw/workspace/MEMORY.md`
   - `~/.openclaw/workspace/HEARTBEAT.md`
   - `~/.openclaw/workspace/TOOLS.md`
   - `~/.openclaw/workspace/IDENTITY.md`
3. 必要时结合本地 OpenClaw docs：
   - `/home/openclaw/.nvm/versions/node/v24.14.0/lib/node_modules/openclaw/docs`
4. 如果用户要“反推配置过程”，优先结合：
   - `~/.bash_history`
   - `journalctl --user -u openclaw-gateway`
   - `openclaw status`
   - `openclaw gateway status`
5. 输出时优先按“配置主题 → 文件位置 → 作用 → 编辑命令 → 修改后操作”组织。
6. 涉及敏感信息时不要明文回传，例如：
   - API key
   - token
   - password
   - 成对设备私密认证材料

## 推荐输出结构

建议使用点式或清单式输出，避免空泛大段叙述。

每个条目尽量包含：
- 配置主题
- 文件/目录路径
- 作用
- 常见可修改项
- 编辑命令（如 `vim ~/.openclaw/openclaw.json`）
- 修改后是否需要：
  - `systemctl --user daemon-reload`
  - `systemctl --user restart openclaw-gateway`
  - `openclaw gateway restart`

## systemd 相关提醒

如果用户环境里同时存在：
- `~/.config/systemd/user/openclaw-gateway.service`
- `/etc/systemd/system/openclaw-gateway.service`

需要明确区分：
- user-level service
- system-level service
- 当前实际生效的是哪一个
- 是否开启了 `loginctl enable-linger <user>`

## Tailscale / Gateway 相关提醒

如果发现 gateway 是 loopback-only（如 127.0.0.1:18789），需要说明：
- 本地监听地址
- 是否通过 Tailscale Funnel / 其他反向入口对外暴露
- `openclaw gateway status` 的结果怎么读

## 不要做的事

- 不要把不存在的配置路径当成已存在事实
- 不要泄露敏感字段明文
- 不要把 user systemd 和 system systemd 混为一谈
- 不要假设所有功能都一定在 `openclaw.json`；有些行为由 workspace 文件决定

## 交付物建议

当用户要求“生成指南 / skill / 笔记”时，优先创建：
- `skills/openclaw-config-guide/SKILL.md`：复用型 skill
- `notes/openclaw-config-guide.md`：面向人的说明文档

如用户还要求“上传笔记”，先确认可用上传方式（例如 workspace 内已有专门 skill），然后再执行上传。