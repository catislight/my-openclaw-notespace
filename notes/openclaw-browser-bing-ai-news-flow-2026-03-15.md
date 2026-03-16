# OpenClaw 源码级笔记：用户要求“用浏览器 Bing 搜索 2026/3/15 的 AI 资讯并总结”时的执行流程

## 一句话结论

这类请求在 OpenClaw 里，本质上不是“调用一个 Bing 搜索 API”，而是：

**聊天消息进入 embedded runner → runner 为本轮 agent run 组装工具与系统提示 → browser tool 暴露给模型 → 模型自主决定发出哪些 browser action（open / snapshot / act 等）→ browser control service 或 sandbox bridge / node proxy 执行浏览器操作 → 页面内容以 tool result 的形式回喂模型 → 模型生成总结文本 → 文本回到当前会话。**

固定的是执行框架；**具体发哪些 browser action，不是仓库里写死的，而是模型根据提示词、工具描述、上下文自己决定的。**

---

## 1. 整体框架：固定的是 runner + tool loop，不固定的是每一步 browser action

当用户发出：

> 用浏览器 Bing 搜索一下 2026/3/15 号的 AI 资讯，总结发给我

系统不会把它解析成某个“search_bing”专用函数。  
OpenClaw 做的是把这条消息放进一次 agent run，让模型在**可用工具集合**中自己选择行动。

所以这里的关键要分开看：

### 1.1 固定部分
固定的是整条 agent 执行链路：

1. 接收用户消息
2. 创建本轮 run
3. 组装 system prompt
4. 暴露工具给模型
5. 模型决定是否用 browser
6. browser tool 执行
7. tool result 回到模型
8. 模型生成最终总结
9. 总结回当前会话

### 1.2 非固定部分
非固定的是模型在中间到底发哪些 browser action，比如：

- 直接 `open` Bing 搜索结果页
- 先打开 Bing 首页，再 `snapshot` + `act(type/click)`
- 在结果页上继续 `snapshot`
- 点进若干文章页继续 `snapshot`
- 必要时做更多 `act`

也就是说：

**“用浏览器 Bing 搜索”在 OpenClaw 里更像一段浏览器自动化工作流，而不是一个写死的检索接口。**

---

## 2. 入口：聊天消息如何进入本轮 agent run

聊天消息进入 embedded runner 后，会进入一次完整的 run 尝试。  
核心准备逻辑在：

- `src/agents/pi-embedded-runner/run/attempt.ts:845-889`
- `src/agents/pi-embedded-runner/run/attempt.ts:1181-1193`

这里会做两件关键事情：

### 2.1 生成本轮可用工具
runner 会调用 `createOpenClawCodingTools(...)` 之类的工具组装逻辑，准备本轮允许模型使用的工具集合。

这一步的意义是：

- 当前用户请求不是裸文本直接给模型回答
- 而是“带工具能力的 session”交给模型
- 模型之后能否调用 browser、read、exec、message 等，取决于这里暴露出去的工具集

### 2.2 创建 agent session
然后 runner 调用 `createAgentSession(...)`，把：

- system prompt
- 会话上下文
- 工具集合
- sandbox / session 信息

一起交给模型。

从源码结构上看，这一步相当于正式把“这次你可以调用哪些工具”以及“当前环境长什么样”告诉模型。

---

## 3. system prompt 如何影响模型选择 browser

这个阶段还有一个很关键但容易忽略的点：  
**模型不是盲选工具，它会被 system prompt 显式告知当前有哪些工具、浏览器能力是否可用。**

相关位置：

- `src/agents/system-prompt.ts:430-469`
- `src/agents/system-prompt.ts:538-545`

这里通常会把类似下面的信息放进 prompt：

- 当前有 browser tool 可用
- sandbox browser 是否可用
- host browser control 是否允许
- 某些 profile 或 browser relay 的使用条件
- 在什么情况下优先用什么浏览器路径

这会直接影响模型决策。

### 3.1 为什么“用浏览器”“Bing”会强烈诱导模型选 browser
如果用户只说：

> 查一下 2026/3/15 的 AI 资讯

模型可能会考虑：

- `web_search`
- `web_fetch`
- `browser`

但如果用户明确说：

> 用浏览器 Bing 搜索一下……

那么这相当于给了模型非常强的操作偏好信号：

- 要“浏览器”
- 要“Bing”
- 不是泛搜索 API
- 更像人工浏览网页再总结

因此模型通常会倾向选 `browser`，而不是 `web_search`。  
这**不是硬编码规则**，而是 prompt + tool 描述 + 用户措辞共同作用下的模型决策结果。

---

## 4. browser tool 是如何暴露给模型的

工具注入的核心路径在：

- `src/agents/openclaw-tools.ts:128-133`
- `src/agents/tools/browser-tool.ts:281-389`

### 4.1 注入点
`createOpenClawTools()` 会把 browser tool 放进 agent 工具集合中。

也就是说，对模型来说，browser 是本轮 session 的“一个可调用工具”。

### 4.2 真正实现
browser 工具的具体实现由 `createBrowserTool()` 提供。  
这个实现负责：

- 定义 tool schema / 可接受参数
- 决定 target / profile / node 的路由逻辑
- 把模型给出的 browser action 映射到底层浏览器控制服务
- 把结果包装成 tool result 回给模型

这一层是“模型工具调用”与“浏览器控制后端”之间的桥。

---

## 5. “Bing 搜索”在源码里不会变成专用 search 动作

源码里没有一个专门叫：

- `search_bing`
- `bing_search`
- `browser.search`

这样的 action。

browser tool 的常见 action 是更通用的浏览器操作，例如：

- `open`
- `snapshot`
- `act`

对应位置例如：

- `open`：`src/agents/tools/browser-tool.ts:421-438`
- `snapshot`：`src/agents/tools/browser-tool.ts:493-505`
- `act`：`src/agents/tools/browser-tool.ts:644-657`

### 5.1 常见路径一：直接打开 Bing 搜索结果页
模型可能最省事地这样做：

1. 直接构造 Bing 搜索 URL
2. 调 `browser action=open`
3. 打开搜索结果页
4. 然后 `snapshot` 读取结果

例如等价于：

- 打开 `https://www.bing.com/search?q=2026%2F3%2F15+AI+资讯`

这种方式适合模型已经知道如何把用户 query 直接编码进 URL。

### 5.2 常见路径二：先打开 Bing 首页，再模拟输入
另一种更“像人操作”的路径是：

1. `open` 打开 Bing 首页
2. `snapshot` 读取页面元素
3. `act kind=type` 往搜索框输入关键词
4. `act kind=click` 点击搜索按钮
5. 再 `snapshot` 读取结果页

这条路线对模型的要求更高，因为它要先理解页面结构，再引用 snapshot 中的元素去做 act。

### 5.3 搜索后通常还会继续做什么
即使到了结果页，也通常不会立刻总结。  
模型一般还会继续：

- `snapshot` 读取搜索结果列表
- 识别值得点开的条目
- `act` 点击结果
- 在文章页再次 `snapshot`
- 从多条来源汇总后再总结

所以“总结”并不是来自 Bing 搜索页本身，而是经常来自**搜索结果页 + 若干点开的页面快照**的组合。

---

## 6. target 和 profile 是怎么选出来的

这一段是 browser tool 的核心路由逻辑之一。  
关键位置：

- `src/agents/tools/browser-tool.ts:251-279`
- `src/agents/tools/browser-tool.ts:286-334`

### 6.1 target 不是永远固定 host
browser tool 不会永远把请求发到宿主机浏览器。  
它会根据当前 session 环境选择目标执行位置：

- `sandbox`
- `host`
- `node`

默认规则大致是：

- **如果当前 session 有 sandbox browser bridge，默认 target 往往是 sandbox**
- **否则默认走 host**

所以默认值其实取决于当前运行环境，而不是全局写死成 host。

### 6.2 profile 也不只一个
browser 配置解析时会自动补一些内建 profile。  
相关逻辑在：

- `src/browser/config.ts:205-318`

内建 profile 通常包括：

- `openclaw`：OpenClaw 自己管理的浏览器
- `chrome` / relay 相关 profile：用于扩展接管当前用户 Chrome 标签页

### 6.3 为什么这类请求通常走 openclaw profile
如果用户只是说：

> 用浏览器 Bing 搜索一下……

没有提到：

- Chrome 扩展
- Browser Relay
- attach tab
- toolbar icon

那么模型通常会用默认 profile，也就是 OpenClaw 自己管理的 profile（常见就是 `openclaw`）。

而当用户明确提到“浏览器扩展接管当前标签页”之类需求时，tool 描述才会强烈引导模型改用 relay / chrome 相关 profile。

### 6.4 node 路径什么时候出现
如果系统配置了具备 browser capability 的 node，browser tool 也可能改走 node 代理路径。  
相关位置：

- `src/agents/tools/browser-tool.ts:131-199`
- `src/agents/tools/browser-tool.ts:336-357`

但对“普通聊天里让我用浏览器 Bing 搜一下”这种请求来说，**主路径通常还是 sandbox 或 host，不是 node**。

---

## 7. 聊天场景下，browser tool 不走 CLI 的 browser.request

这是理解源码时特别容易混淆的一点。

用户在聊天里发请求，并不是走：

- `openclaw browser ...`
- `browser.request` RPC

聊天场景更接近：

**模型调用 browser tool → browser tool 内部调用 browser client → browser client 连接 browser control service**

而 CLI 的 `browser.request` 更偏向：

- 命令行子命令
- 显式调用浏览器控制接口
- 手工运维 / 调试路径

相关位置：

- `src/cli/browser-cli-shared.ts:30-62`
- `src/gateway/server-methods/browser.ts:142-265`

所以从调用链角度要明确区分：

### 7.1 聊天里的主链
`browser tool.execute()`
→ `browserOpenTab / browserSnapshot / browserAct`
→ `fetchBrowserJson()`

### 7.2 CLI 的另一条链
`openclaw browser ...`
→ `browser.request`
→ gateway server method

这两条链底层可能共享一部分 browser control 服务，但入口不一样。

---

## 8. host / sandbox / node 三条执行分支

真正把 browser action 发下去时，核心分流点在 `fetchBrowserJson()`。  
相关位置：

- `src/browser/client-fetch.ts:190-279`

### 8.1 host 分支
如果 `baseUrl` 为空，表示目标是本地 host。  
这时会：

- 确保 browser control service 可用
- 在进程内或本地路径直接 dispatch 浏览器路由

也就是“本机本地控制”的模式。

相关位置：

- `src/browser/client-fetch.ts:190-279`
- `src/browser/control-service.ts:23-52`

### 8.2 sandbox bridge 分支
如果 `baseUrl` 是类似 `http://127.0.0.1:...` 的地址，说明当前走的是 sandbox bridge。  
这时 browser action 会转成带鉴权的 HTTP 请求发往 sandbox 内的浏览器桥接服务。

这允许模型在隔离环境里完成浏览器操作，而不是直接碰宿主环境。

### 8.3 node 分支
如果前面路由逻辑已经决定走 node，那么就不会走上面这条 `fetchBrowserJson()` 本地/bridge 路径，而是改成：

- `node.invoke("browser.proxy")`

也就是让远端 node 代为执行 browser 操作。

---

## 9. open 动作到底怎么把浏览器拉起来

以最常见的 `openclaw` profile 为例，打开标签页会先经过 `/tabs/open` 路由。  
相关位置：

- `src/browser/routes/tabs.ts:119-135`

### 9.1 ensureBrowserAvailable()
在真正打开 tab 之前，系统先确保这个 profile 对应的浏览器可用。  
核心逻辑在：

- `src/browser/server-context.availability.ts:139-237`

这里会区分几种浏览器来源模式：

#### 9.1.1 local-managed
本地托管浏览器模式。  
如果检测到 CDP（Chrome DevTools Protocol）不可达，就会尝试本地启动一个托管浏览器。

也就是说：

- OpenClaw 自己管浏览器进程
- 自己维护 user-data-dir
- 自己确保 CDP 可连

#### 9.1.2 local-extension-relay
本地扩展 relay 模式。  
这时通常**不会直接启动浏览器本体**，而是确保 relay server 在运行。

适合“接管用户已打开的浏览器标签页”。

#### 9.1.3 remote-cdp
远程 CDP 模式。  
这时主要是检查远端的调试入口是否可连，而不是在本机起浏览器。

### 9.2 本地托管浏览器如何启动
本地启动逻辑主要在：

- `src/browser/chrome.ts:238-405`

`launchOpenClawChrome()` 之类的逻辑会做这些事：

1. 在系统上寻找可用浏览器可执行文件  
   可能是 Chrome / Chromium / Brave / Edge 等。

2. 为 profile 准备独立的 `user-data-dir`  
   这样不同 profile 的 cookie、会话、缓存相互隔离。

3. 增加 `--remote-debugging-port`  
   这是后续 Playwright / CDP 控制浏览器的基础。

4. 必要时创建一个初始空白页  
   例如 `about:blank`，便于后续 tab 管理。

### 9.3 真正打开 Bing 搜索页
浏览器可用之后，`openTab()` 会真正导航到 Bing 搜索页。  
相关位置：

- `src/browser/server-context.tab-ops.ts:134-223`

这里不仅是“打开 URL”，通常还会受：

- SSRF guard
- navigation guard
- 目标 URL 安全校验

这些安全机制约束，避免工具被滥用去访问不该访问的内部地址。

---

## 10. 页面内容是怎么“读回来”给模型的

这是整个流程最核心的一步。  
因为模型最终要总结，必须先拿到页面内容。

这个过程主要不是 OCR，而是依靠 **snapshot**。

### 10.1 /snapshot 路由
相关位置：

- `src/browser/routes/agent.snapshot.ts:212-320`

`snapshot` 的职责是把当前页面转成适合模型消费的结构化内容。  
根据环境与能力不同，可能采用：

- Playwright AI snapshot
- role snapshot
- aria snapshot

### 10.2 为什么不是单纯截图
截图只是视觉像素，模型要做高质量网页理解，结构化快照通常更有用。  
snapshot 能返回：

- 页面元素树
- 可交互元素引用
- 标题、链接、按钮、搜索框等语义信息
- 页面文本内容

所以模型不仅能“读懂”页面，还能在下一步 act 时引用元素。

### 10.3 snapshot 结果如何包装回模型
browser tool 在拿到 snapshot 结果后，不是裸文本直接塞给模型，而是封装成 tool result。  
相关位置：

- `src/agents/tools/browser-tool.actions.ts:107-239`

其中一个关键点是，它会通过类似 `wrapExternalContent(...)` 的逻辑，把页面内容标成：

- source: `"browser"`
- 外部不可信内容

这一步很重要，因为网页内容本质上属于外部输入，不能直接当系统指令使用。  
也就是说，OpenClaw 在设计上明确把网页内容视作**不可信外部数据**。

---

## 11. act 如何驱动后续交互

如果模型只 snapshot 一次不够，它还可以继续发 `act`。

相关位置：

- `src/browser/routes/agent.shared.ts:103-147`
- `src/browser/routes/agent.act.ts:25-310`

### 11.1 act 的典型能力
`act` 可以做的事情包括：

- click
- type
- press
- select
- wait
- evaluate
- hover
- drag
- fill

这使得模型能像一个会操作浏览器的用户一样继续工作。

### 11.2 act 的底层依赖
底层通常通过 Playwright 和 CDP 与浏览器对接，关键连接点在：

- `src/browser/pw-session.ts:332-365`

这里常见是通过：

- `chromium.connectOverCDP(...)`

接入已有浏览器实例。

这意味着 browser tool 的“自动化”不是浏览器扩展脚本随便乱点，而是依靠比较标准、稳定的浏览器自动化基础设施。

---

## 12. 最终总结是怎么回到当前会话的

当 browser tool 把若干 snapshot / act 的结果回喂给模型后，模型就会在当前上下文中生成总结文本。

这里通常**不会额外再走一次 `message` tool**，原因很简单：

- 用户本来就在当前对话里提问
- 最自然的交付方式就是当前 assistant 直接回复
- 不需要再主动发一条新消息到别处

相关位置：

- `src/agents/pi-embedded-runner/run/attempt.ts:1512-1536`

这里 runner 会把模型的最终输出流式绑定回当前 run / 当前会话。  
也就是：

**browser tool 负责拿信息，assistant 最终文本负责交付结果。**

---

## 13. 把整个例子串成一条最典型的时序

以下是一条最常见、最符合源码结构的执行链：

1. 用户发消息：  
   “用浏览器 Bing 搜索一下 2026/3/15 号的 AI 资讯，总结发给我”

2. embedded runner 启动本轮 run  
   - 组装 system prompt  
   - 组装工具集  
   - 创建 agent session

3. system prompt 告诉模型：  
   当前 browser 工具可用，当前环境允许哪种 browser path

4. 模型判断：  
   由于用户明确要求“用浏览器”“Bing”，优先使用 browser tool

5. browser tool 决定 target/profile  
   - 常见是 `target=sandbox` 或 `target=host`  
   - 常见 profile 是 `openclaw`

6. 模型发出 `browser action=open`  
   直接打开 Bing 搜索 URL，或者先打开 Bing 首页

7. `/tabs/open` 确保浏览器可用  
   必要时启动本地托管浏览器并建立 CDP 控制能力

8. 页面打开后，模型发 `browser action=snapshot`  
   读取搜索结果页的结构化内容

9. 如有必要，模型继续发 `browser action=act`  
   点击若干结果、输入补充关键词、翻页、等待加载等

10. 后续页面继续 `snapshot`  
    文章内容与页面结构作为 tool result 回到模型上下文

11. 模型综合多个页面内容  
    生成中文总结

12. runner 将最终 assistant 文本返回当前会话  
    用户收到总结结果

---

## 14. 这个流程里最关键的源码级判断

如果要把这套原理压缩成最关键的四句话，就是：

### 14.1 聊天入口是 browser tool，不是 CLI browser.request
聊天中的 browser 使用，核心是 agent tool 调用，而不是命令行子命令路径。

### 14.2 “Bing 搜索”不是专用接口，而是浏览器自动化
没有一个写死的 Bing 搜索 API action，模型是在通用浏览器动作上拼出搜索流程。

### 14.3 页面内容回到模型靠 snapshot
模型不是凭空总结，也不是单靠截图 OCR，而是依赖结构化 snapshot / Playwright 页面抽取结果。

### 14.4 最后的“发给我”通常就是当前 assistant 回复
信息拿到后，模型直接在当前聊天里输出总结文本，而不是再走额外消息发送链路。

---

## 15. 这套设计为什么合理

从架构设计角度看，这样做有几个明显优点：

### 15.1 通用性高
不需要为 Bing、Google、知乎、GitHub、新闻站分别写专用搜索工具。  
只要浏览器能打开，模型就能操作。

### 15.2 更符合自然语言请求
用户只要说“去网页上看一下”“用浏览器查一下”，模型就能自己规划操作步骤。

### 15.3 更容易扩展到复杂任务
同样的 browser tool 不仅能搜索，还能：

- 登录页面
- 填表
- 点按钮
- 截取 UI
- 读取复杂交互页面

### 15.4 安全边界更清晰
通过：

- target / profile 路由
- sandbox bridge
- node proxy
- external content 包装
- SSRF / navigation guard

系统把浏览器操作限制在可控边界里。

---

## 16. 一些容易误解的细节

### 16.1 “模型决定发哪些 browser action” ≠ 完全无约束
虽然具体动作是模型决定的，但它仍受这些因素约束：

- tool schema
- system prompt
- target / profile 规则
- route 安全限制
- browser control service 能力边界

### 16.2 summary 不是某个固定 summarizer 在做
不是“浏览器工具自己总结”，而是：
- browser tool 负责拿页面内容
- 主模型负责理解、筛选、归纳、生成总结

### 16.3 snapshot 内容不是可信系统输入
网页文本会被明确标记为外部不可信内容，防止网页 prompt injection 直接污染系统层逻辑。

### 16.4 并不是每次都会点开很多页面
如果搜索结果页本身已经足够密集，模型可能只 snapshot 结果页就总结。  
但如果问题更严谨，通常会点开几条结果再综合。

---

## 17. 最后的总结

对于“用浏览器 Bing 搜索一下 2026/3/15 的 AI 资讯，总结发给我”这种请求，OpenClaw 的本质机制可以概括为：

**它不是调用一个写死的 Bing 搜索函数，而是把请求交给一个拥有 browser tool 的 agent run。模型依据 system prompt 和工具描述，自主决定浏览器动作；browser control service / sandbox bridge / node proxy 执行这些动作；页面内容通过 snapshot 等方式回流为 tool result；最后模型再基于这些外部页面内容生成总结并回复当前会话。**

所以从源码级理解，这套机制的核心关键词是：

- embedded runner
- tool injection
- browser tool
- target/profile routing
- browser control service
- snapshot as structured page ingestion
- tool result feedback loop
- final assistant response in current session
