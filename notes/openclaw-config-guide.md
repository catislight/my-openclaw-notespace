# OpenClaw 配置指南

这份笔记基于当前机器上的实际 OpenClaw 部署结构整理，适合用于：
- 快速理解目录结构
- 找到每类配置的落点
- 知道哪些文件可以手改
- 清楚改完之后该 reload / restart 哪一层
- 回顾 gateway / systemd / devices / workspace / tailscale 的关系

- 主配置根目录：`~/.openclaw/`
  - 这是 OpenClaw 的主数据目录。
  - 常见内容包括：
    - `openclaw.json`：主配置
    - `.env`：环境变量
    - `devices/`：设备配对信息
    - `workspace/`：agent 工作区
    - `cron/`：定时任务相关数据
    - `credentials/` 或 workspace 下的 credentials：某些凭证材料

- 主配置文件：`~/.openclaw/openclaw.json`
  - 这是最核心的配置文件。
  - 常见配置主题通常都在这里：
    - `models`：模型、provider、默认模型
    - `gateway`：网关监听、端口、远程接入、认证
    - `tools`：工具 profile
    - `extensions / plugins`：扩展、插件开关、策略
    - `session`：会话范围、DM scope 等
    - `agent`：workspace、压缩策略等
    - 各消息渠道/集成配置（例如 Feishu）
  - 编辑命令：
    - `vim ~/.openclaw/openclaw.json`
    - `sudo vim /home/openclaw/.openclaw/openclaw.json`
  - 修改后：
    - 某些配置会被运行中的 gateway 检测到并触发重启
    - 稳妥做法是重启 gateway：
      - `systemctl --user restart openclaw-gateway`
      - 或你自己的包装命令：`openclaw-userctl restart openclaw-gateway`

- 环境变量文件：`~/.openclaw/.env`
  - 用于放不想直接写进 JSON 的环境变量。
  - 是否生效取决于谁在读取它：
    - 当前 system-level unit 会读它（如果 unit 里有 `EnvironmentFile=`）
    - user-level unit 是否读取，要看 unit 文件有没有显式写 `EnvironmentFile=`
  - 编辑命令：
    - `vim ~/.openclaw/.env`
  - 修改后：
    - 如果 systemd unit 引用了它，要重启对应 service

- 配置备份文件：`~/.openclaw/openclaw.json.bak*`
  - 用于回看历史状态、做 diff、回滚参考。
  - 常见命令：
    - `ls -l ~/.openclaw/openclaw.json*`
    - `diff -u ~/.openclaw/openclaw.json.bak ~/.openclaw/openclaw.json`

- 设备配置目录：`~/.openclaw/devices/`
  - 用于设备引导、配对、授权状态记录。
  - 常见文件：
    - `bootstrap.json`：bootstrap token
    - `paired.json`：已配对设备
    - `pending.json`：待配对设备

- `~/.openclaw/devices/bootstrap.json`
  - 存 bootstrap token。
  - 通常是设备首次引导或临时连接时会用到。
  - 编辑命令：`vim ~/.openclaw/devices/bootstrap.json`
  - 注意：里面通常有敏感 token，不要随意外发。

- `~/.openclaw/devices/paired.json`
  - 记录已经配对成功的设备。
  - 可查看设备类型、角色、授权 scope 等。
  - 编辑命令：`vim ~/.openclaw/devices/paired.json`
  - 风险提示：手改容易破坏配对关系，通常更适合先只读查看。

- `~/.openclaw/devices/pending.json`
  - 记录还在等待确认/完成的设备。
  - 编辑命令：`vim ~/.openclaw/devices/pending.json`

- user-level systemd unit：`~/.config/systemd/user/openclaw-gateway.service`
  - 这是当前机器上真正生效的 gateway 托管文件。
  - 作用：
    - 定义 gateway 如何启动
    - 是否自动重启
    - 启动命令、环境变量、目标 target
  - 常见可改项：
    - `ExecStart=`
    - `Restart=`
    - `RestartSec=`
    - `Environment=`
    - `EnvironmentFile=`
    - `WorkingDirectory=`
  - 编辑命令：
    - `vim ~/.config/systemd/user/openclaw-gateway.service`
  - 修改后要执行：
    - `systemctl --user daemon-reload`
    - `systemctl --user restart openclaw-gateway`

- user-level systemd 备份：`~/.config/systemd/user/openclaw-gateway.service.bak`
  - 旧版本备份，适合对比，不建议当主配置改。

- system-level systemd unit：`/etc/systemd/system/openclaw-gateway.service`
  - 这是系统级 gateway unit。
  - 当前机器上它存在，但不是当前生效主路径。
  - 编辑命令：
    - `sudo vim /etc/systemd/system/openclaw-gateway.service`
  - 修改后要执行：
    - `sudo systemctl daemon-reload`
    - `sudo systemctl restart openclaw-gateway`
  - 注意：system-level 和 user-level 可能同时存在，排障时必须区分。

- user systemd 可用性的前置条件
  - 如果你要依赖 `systemctl --user` 在无登录会话环境下常驻运行，通常需要：
    - `sudo loginctl enable-linger openclaw`
  - 检查命令：
    - `loginctl show-user openclaw -p Linger -p RuntimePath -p State`
  - 如果遇到：
    - `Failed to connect to bus: $DBUS_SESSION_BUS_ADDRESS and $XDG_RUNTIME_DIR not defined`
    - 说明当前命令执行环境没有 user bus，需要补环境或改用包装命令。

- 推荐的 user systemd 包装命令
  - 为避免每次手动 export：
    - 创建 `/usr/local/bin/openclaw-userctl`
  - 样例：
    - `openclaw-userctl status openclaw-gateway`
    - `openclaw-userctl restart openclaw-gateway`
    - `openclaw-userctl daemon-reload`

- Gateway 配置主题
  - 配置落点：主要在 `~/.openclaw/openclaw.json`
  - 常见内容：
    - 监听地址 `bind`
    - 端口 `port`
    - 认证 `auth`
    - 远程入口 / Tailscale 相关项
    - 是否允许某些远程访问方式
  - 当前机器的典型状态是：
    - loopback only（127.0.0.1）
    - 端口 18789
  - 检查命令：
    - `openclaw gateway status`
    - `ss -lntp | grep 18789`

- Tailscale / Funnel 配置主题
  - 配置和运行状态通常涉及两部分：
    - `openclaw.json` 里的 gateway/tailscale 相关项
    - Tailscale 系统当前状态
  - 常用命令：
    - `tailscale status`
    - `tailscale ip`
    - `tailscale funnel status`
    - `sudo tailscale funnel reset`
    - `sudo tailscale funnel --bg --yes 18789`
  - 思路通常是：
    - gateway 本地只监听 127.0.0.1:18789
    - 通过 Tailscale Funnel 暴露对外入口

- 模型与 Provider 配置主题
  - 主要落点：`~/.openclaw/openclaw.json`
  - 常见内容：
    - `models.mode`
    - `models.providers`
    - provider 的 `baseUrl`
    - `apiKey`
    - provider 类型 / API 类型
    - 默认模型、显示名、模型 id
  - 风险提示：
    - API key 可能直接位于此文件
    - 回传/截图时不要明文泄露

- 插件 / 扩展配置主题
  - 主要落点：`~/.openclaw/openclaw.json`
  - 常见内容：
    - `plugins.entries.*`
    - `extensions.*`
    - 各插件 `enabled`
    - allowlist / group policy / tool 注册行为
  - 如果接了 Feishu 之类渠道，相关启用状态一般也在这里。

- Session / Agent 配置主题
  - 主要落点：`~/.openclaw/openclaw.json`
  - 常见内容：
    - `session.*`
    - `agent.workspace`
    - `agent.compaction`
    - 工具 profile
  - 这些影响会话范围、压缩策略、工作区定位等。

- Workspace 目录：`~/.openclaw/workspace/`
  - 这是 agent 的长期工作区。
  - 里面既有“人设/记忆类”文件，也可能有技能、文档、脚本、临时项目。

- `~/.openclaw/workspace/AGENTS.md`
  - 定义 agent 的全局工作约定。
  - 会影响：
    - 启动流程
    - 红线规则
    - 心跳策略
    - 对外/对内动作边界
  - 编辑命令：`vim ~/.openclaw/workspace/AGENTS.md`

- `~/.openclaw/workspace/SOUL.md`
  - 定义 agent 的人格、语气、价值倾向。
  - 编辑命令：`vim ~/.openclaw/workspace/SOUL.md`

- `~/.openclaw/workspace/USER.md`
  - 记录用户称呼、偏好、上下文备注。
  - 编辑命令：`vim ~/.openclaw/workspace/USER.md`

- `~/.openclaw/workspace/MEMORY.md`
  - 长期记忆。
  - 适合沉淀稳定偏好、决策、长期背景。
  - 编辑命令：`vim ~/.openclaw/workspace/MEMORY.md`

- `~/.openclaw/workspace/memory/YYYY-MM-DD.md`
  - 每日记忆 / 日志。
  - 适合记录当天发生的事、短期上下文。
  - 编辑命令示例：
    - `vim ~/.openclaw/workspace/memory/2026-03-15.md`

- `~/.openclaw/workspace/HEARTBEAT.md`
  - 定义 heartbeat 轮询时的检查项。
  - 编辑命令：`vim ~/.openclaw/workspace/HEARTBEAT.md`

- `~/.openclaw/workspace/TOOLS.md`
  - 记录本机私有工具备注，比如 SSH 主机、设备别名、TTS 偏好等。
  - 编辑命令：`vim ~/.openclaw/workspace/TOOLS.md`

- `~/.openclaw/workspace/IDENTITY.md`
  - 记录 agent 的名字、形象、emoji、vibe。
  - 编辑命令：`vim ~/.openclaw/workspace/IDENTITY.md`

- `~/.openclaw/workspace/BOOTSTRAP.md`
  - 首次初始化引导。
  - 一般初始化完成后可删除或不再依赖。
  - 编辑命令：`vim ~/.openclaw/workspace/BOOTSTRAP.md`

- `~/.openclaw/workspace/skills/`
  - 存放本地 skill。
  - 每个 skill 通常至少有一个 `SKILL.md`。
  - 适合放：
    - 可复用操作指南
    - 某类任务的标准步骤
    - 本机专用技能
  - 示例：
    - `~/.openclaw/workspace/skills/openclaw-config-guide/SKILL.md`

- `~/.openclaw/workspace/notes/`
  - 适合放面向人的说明笔记、运维手册、排障记录。
  - 这次整理的文档就适合放在这里。

- `~/.openclaw/workspace/cron/`
  - 与 cron / 定时任务相关的工作区内容。
  - 如果有精确调度任务，可能在这里或 OpenClaw 的定时机制里体现。

- `~/.openclaw/workspace/credentials/`
  - 用于放一些工作区级别凭证材料。
  - 编辑前先明确文件用途，不建议随意手改。

- `~/.openclaw/workspace/completions/`
  - 常见于任务输出、完成记录、历史结果缓存。

- `~/.openclaw/workspace/canvas/`
  - 可能用于中间文档、草稿、画布式内容。

- `~/.openclaw/workspace/agents/`
  - 可能包含子 agent / 相关运行数据或本地代理工作文件。

- 修改后怎么判断该重启哪一层
  - 改 `openclaw.json`：优先重启 gateway
    - `systemctl --user restart openclaw-gateway`
  - 改 user systemd unit：
    - `systemctl --user daemon-reload`
    - `systemctl --user restart openclaw-gateway`
  - 改 system systemd unit：
    - `sudo systemctl daemon-reload`
    - `sudo systemctl restart openclaw-gateway`
  - 改 workspace 下人格/记忆/说明文件：
    - 一般不需要 systemd 重启
    - 但会影响后续 agent 行为

- 常用查看命令
  - `openclaw status`
  - `openclaw gateway status`
  - `systemctl --user status openclaw-gateway`
  - `journalctl --user -u openclaw-gateway -f`
  - `jq . ~/.openclaw/openclaw.json`
  - `jq . ~/.openclaw/devices/paired.json`
  - `find ~/.openclaw -maxdepth 2 -type f | sort`

- 风险点 / 易混点
  - `openclaw.json` 里可能包含敏感字段，不要明文外传
  - user-level 和 system-level `openclaw-gateway.service` 可能同时存在，但不一定同时生效
  - `paired.json` / `bootstrap.json` 手改风险较高，优先只读
  - `.env` 是否被读取取决于当前 unit 是否显式引用
  - workspace 文件会显著影响 agent 行为，但它们不等同于底层 daemon 配置

- 最值得记住的几个编辑入口
  - `vim ~/.openclaw/openclaw.json`
  - `vim ~/.config/systemd/user/openclaw-gateway.service`
  - `vim ~/.openclaw/.env`
  - `vim ~/.openclaw/workspace/AGENTS.md`
  - `vim ~/.openclaw/workspace/SOUL.md`
  - `vim ~/.openclaw/workspace/USER.md`
  - `vim ~/.openclaw/devices/paired.json`

- 一套最常用的配套命令
  - `openclaw status`
  - `openclaw gateway status`
  - `systemctl --user daemon-reload`
  - `systemctl --user restart openclaw-gateway`
  - `journalctl --user -u openclaw-gateway -f`
  - `tailscale funnel status`
