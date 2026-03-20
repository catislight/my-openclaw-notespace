# OpenClaw 远程连接配置流程和要点

## 一句话结论

OpenClaw 的远程连接最佳实践是：**只保留一个远端 Gateway 主实例，默认绑定在 loopback（127.0.0.1:18789）；客户端通过 SSH 隧道或 Tailscale 接入；优先避免直接暴露公网端口。**

---

## 1. 整体架构

- Gateway 是 OpenClaw 的中心节点，负责：
  - 会话状态
  - agent 执行
  - 渠道接入
  - 认证与配置
- 远程访问的核心思路是：
  - **Gateway 跑在固定主机上**
  - 客户端、macOS App、移动节点都去连接这个 Gateway
- 节点本身不是 Gateway。
- 一般情况下，**不要在多台机器上同时各跑一套 Gateway**，除非你明确要做隔离环境。

---

## 2. 核心设计原则

### 2.1 Gateway 默认只绑定本机回环地址
- 默认 WebSocket 绑定在：
  - `127.0.0.1:18789`
- 这意味着 Gateway 默认不直接暴露给局域网或公网。

### 2.2 远程访问通过“转发”或“Tailnet”实现
远程访问通常通过以下方式完成：

- SSH Tunnel
- Tailscale Serve
- Tailscale 直接绑定 tailnet IP
- Tailscale Funnel（公网访问，不推荐作为默认方案）

### 2.3 推荐安全姿势
官方文档的安全倾向很明确：

- **首选：loopback + SSH tunnel**
- **次选：loopback + Tailscale Serve**
- **谨慎：bind tailnet + token/password**
- **最谨慎：Funnel + password，仅在必须公网访问时使用**

---

## 3. 标准远程连接流程

### 第一步：在远端主机运行 Gateway
推荐配置思路：

- `gateway.bind: "loopback"`
- 端口默认 `18789`

这样 Gateway 只监听本机回环地址，不直接对外暴露。

### 第二步：配置服务端认证
需要为 Gateway 配置认证，常见方式：

- token
- password

注意区分两个概念：

- `gateway.auth.*`：**服务端认证配置**
- `gateway.remote.*`：**客户端连接远端时使用的默认凭证来源**

这两个不是一回事，不能混用。

### 第三步：建立远程访问通道
可选两种主流方式：

#### 方式 A：SSH 隧道
把本地端口转发到远端 Gateway 的 loopback 端口。

典型效果：
- 本地连接 `ws://127.0.0.1:18789`
- 实际经 SSH 转发到远端机器的 `127.0.0.1:18789`

#### 方式 B：Tailscale
通过 tailnet 提供远程访问能力，可以选择：

- Serve：仍保持 Gateway 为 loopback
- tailnet bind：Gateway 直接监听 tailnet IP
- Funnel：公网暴露（风险更高）

### 第四步：客户端接入 Gateway
不同模式下，连接地址不同：

#### SSH Tunnel 模式
客户端仍连接本地地址：
- `ws://127.0.0.1:18789`

#### Tailnet 直接绑定模式
客户端连接 Tailnet IP：
- `ws://<tailscale-ip>:18789`
- `http://<tailscale-ip>:18789/`

#### Tailscale Serve 模式
通常通过：
- `https://<magicdns>/`

### 第五步：配置 CLI 远程默认目标（可选）
CLI 可以设置 remote 模式，让后续命令默认打到远端 Gateway，例如配置：

- `gateway.mode: "remote"`
- `gateway.remote.url`
- `gateway.remote.token`

这样本地 CLI 会默认走远端目标。

---

## 4. SSH 远程方案要点

SSH 是 OpenClaw 最通用、最稳妥的远程访问方案，尤其适合：

- macOS App
- 命令行远程运维
- 不希望暴露 Gateway 网络端口的场景

### 典型做法
在本地与远端之间建立如下转发：

- 本地：`127.0.0.1:18789`
- 远端：`127.0.0.1:18789`

然后客户端继续像连本地一样连接 Gateway。

### macOS App 的工作方式
OpenClaw.app 的 remote 模式本质上就是：

- App 连接本地 `ws://127.0.0.1:18789`
- SSH 隧道把流量转发到远端 Gateway

### 文档给出的典型配置步骤
1. 配置 `~/.ssh/config`
2. 设置 `LocalForward 18789 127.0.0.1:18789`
3. 配置 SSH key 免密登录
4. 设置 `OPENCLAW_GATEWAY_TOKEN`
5. 启动 SSH 隧道
6. 重启 OpenClaw.app

### 长期使用建议
如果是 macOS 长期使用，建议用 LaunchAgent：
- 开机自动启动
- 崩溃自动重连
- 后台常驻

这基本是 macOS 远程使用 OpenClaw 的最佳实践。

---

## 5. Tailscale 方案要点

Tailscale 提供了比纯 SSH 更适合“长期在线远端主机”的接入方式。

### 5.1 Serve 模式（推荐）
配置思路：
- `gateway.bind: "loopback"`
- `tailscale.mode: "serve"`

特点：
- Gateway 仍只监听 `127.0.0.1`
- Tailscale Serve 提供 HTTPS、路由和身份头
- 安全性比直接监听 tailnet IP 更好
- 适合 VPS、家庭服务器等长期在线主机

额外说明：
- Control UI / WebSocket 可以利用 Tailscale identity headers
- 但 HTTP API 端点仍然需要 token/password

### 5.2 直接绑定 Tailnet IP
配置思路：
- `gateway.bind: "tailnet"`
- `auth.mode: "token"` 或 `password`

特点：
- Gateway 直接监听 Tailnet IP
- 连接更直接
- 但暴露面更大一些
- 必须开启认证

连接方式：
- `http://<tailscale-ip>:18789/`
- `ws://<tailscale-ip>:18789`

注意：
- 这种模式下，`127.0.0.1:18789` 不再适合作为访问入口。

### 5.3 Funnel 模式（公网访问）
配置思路：
- `gateway.bind: "loopback"`
- `tailscale.mode: "funnel"`
- `auth.mode: "password"`

特点：
- 通过公网 HTTPS 暴露访问入口
- 风险最高
- 文档明确要求必须使用 password 认证

适用场景：
- 只有在确实需要公网接入时才考虑
- 不适合作为默认部署方式

---

## 6. 认证与凭证的关键区别

这是文档里很容易被忽略、但实际最容易踩坑的部分。

### 6.1 服务端认证 vs 客户端凭证
必须区分：

#### 服务端认证
- `gateway.auth.token`
- `gateway.auth.password`

作用：
- 控制谁可以访问 Gateway

#### 客户端默认远端凭证
- `gateway.remote.token`
- `gateway.remote.password`

作用：
- 客户端在 remote 模式下默认拿什么凭证去连接远端

**重点：`gateway.remote.*` 不会自动替代 `gateway.auth.*` 成为服务端鉴权配置。**

### 6.2 显式传参优先级最高
如果使用 CLI 显式传入：
- `--token`
- `--password`

那么它们优先级最高。

### 6.3 使用 `--url` 时要显式带凭证
文档特别强调：

如果 CLI 命令里手动传了 `--url`，那么不会自动复用配置文件或环境变量里的隐式凭证。

也就是说：
- 传了 `--url`
- 就要一起显式传 `--token` 或 `--password`

否则可能会直接认证失败。

### 6.4 非 loopback 绑定必须开认证
如果 Gateway 绑定在以下类型上：
- `lan`
- `tailnet`
- `custom`
- 或某些情况下的 `auto`

都必须启用 token 或 password 认证，不能裸露。

### 6.5 TLS 指纹固定
如果使用 `wss://`，可以配置：
- `gateway.remote.tlsFingerprint`

作用：
- 固定远端证书指纹
- 提高远程 TLS 连接的安全性

---

## 7. 安全建议总结

OpenClaw 文档对安全的建议非常明确：

### 推荐做法
- Gateway 尽量只绑定 loopback
- 远程访问通过 SSH 或 Tailscale 完成
- 尽量不要直接对公网暴露 Gateway
- 非 loopback 模式一定要启用认证

### 明文 WS 的限制
- `ws://` 默认适合 loopback 使用
- 私有网络里如果硬要使用不安全 WS，需要显式 break-glass 配置

### 关于 Tailscale 身份头
在 `tailscale.mode = "serve"` 且 `gateway.auth.allowTailscale = true` 时：
- Control UI / WebSocket 可以基于 Tailscale 身份头完成认证
- 但 HTTP API 仍需要 token/password

如果你不希望依赖这种 tokenless 认证：
- 关闭 `gateway.auth.allowTailscale`
- 或强制使用 `password` 模式

### 浏览器控制的安全建议
如果 Gateway 在一台机器上，但浏览器在另一台机器上：
- 应该在浏览器所在机器运行 node host
- 让 Gateway 通过 Gateway WS 代理调用 node
- 不需要单独暴露浏览器控制服务

文档特别建议：
- 浏览器控制应视为高权限能力
- 避免使用 Funnel
- 尽量保持 tailnet-only + 明确节点配对

---

## 8. 常见部署场景建议

### 场景 1：希望 OpenClaw 24 小时在线
推荐：
- 把 Gateway 部署在 VPS 或家庭服务器
- 使用 Tailscale Serve 或 SSH 接入

优点：
- 主机稳定在线
- laptop 睡眠不会影响 OpenClaw 可用性

### 场景 2：家里台式机跑 Gateway，外面笔记本远程控制
推荐：
- macOS App Remote over SSH

优点：
- 使用体验顺畅
- 安全性高
- 本地 App 仍像操作本地一样工作

### 场景 3：笔记本自己跑 Gateway，但偶尔需要其他设备访问
推荐：
- SSH tunnel
- 或 Tailscale Serve

不推荐：
- 为了省事直接暴露 Gateway 到公网

---

## 9. 最佳实践建议

如果目标是“稳定、安全、低维护成本”的远程连接，推荐直接使用下面这个组合：

### 推荐方案
- 远端主机运行唯一 Gateway
- `gateway.bind: "loopback"`
- 配置 `gateway.auth.token` 或 `password`
- 有 Tailscale 就优先用 `tailscale.mode: "serve"`
- macOS App 场景优先用 SSH tunnel
- CLI 场景可配置 `gateway.mode: "remote"`

### 不推荐方案
- 直接把 Gateway 暴露到公网
- 在不清楚认证配置的情况下使用非 loopback bind
- 混淆 `gateway.auth.*` 与 `gateway.remote.*`
- 把 Funnel 当成默认部署方式

---

## 10. 最终总结

OpenClaw 远程连接的本质不是“把 Gateway 直接暴露出去”，而是：

1. 在固定主机上运行唯一 Gateway
2. 默认只绑定 loopback
3. 使用 SSH 或 Tailscale 作为远程接入层
4. 用 token/password 做认证
5. 只有确有必要时才考虑更开放的暴露方式

一句话概括：

**OpenClaw 的远程接入最佳实践，就是“单一远端 Gateway + loopback 绑定 + SSH/Tailscale 接入 + 明确认证 + 尽量不公网暴露”。**
