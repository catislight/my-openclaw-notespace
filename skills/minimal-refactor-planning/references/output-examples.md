# Output Examples

Use these examples to keep responses implementation-oriented, concrete, and scoped.

---

## Example 1: Generic minimal-change plan

I’d use the smallest change that centralizes the shared flow without widening scope.

### Options
1. Keep the current structure and patch each call site.  
   Fastest short-term, but duplication stays and drift gets worse.

2. Introduce one shared owner for the core flow.  
   Best fit here: one place owns the logic, call sites become thin triggers.

3. Split into several new layers now.  
   More flexible in theory, but too much abstraction for this change.

### Recommendation
Go with option 2.

### Design
- Add one shared module/service/hook to own the core flow
- Keep current entry points, but reduce them to triggering and param passing
- Preserve current success/error semantics

### Flow
1. Entry point receives user/system input
2. Shared owner validates and runs the main flow
3. Result is returned to the caller
4. Caller handles display/response only

### Change scope
- Move duplicated logic out of existing entry points
- Add one shared owner
- Update callers to delegate to it
- Remove temporary glue/indirect wiring where possible

### Validation
- Verify extracted pure logic
- Run targeted regression on the main path
- Run local lint/typecheck/build as appropriate

---

## Example 2: Frontend-style answer

我建议这次先做最小收口：把共享流程提到共同上层持有，两个入口都只保留触发。

### 方案对比
1. 继续让两个组件各自维护流程  
   改动最少，但重复逻辑还在，后面还会继续漂。

2. 把流程收进一个共享 owner  
   这次最合适，状态和副作用都能统一，入口只负责触发。

3. 一次拆成多个 hook / service  
   不是不能做，但对这次来说偏重了。

### 推荐方案
新增一个共享层承接主流程，页面负责持有，子组件只接触发器和必要状态。

### 数据流
1. 页面初始化共享能力
2. 不同入口调用同一个 trigger
3. 共享层执行主流程
4. 成功后统一回调
5. 失败后统一错误处理

### 改动范围
- 从组件 A 删除内置流程
- 从组件 B 删除事件式触发
- 新增共享抽象
- 页面层负责把能力传给两个入口

### 验证
- 核心辅助逻辑定向验证
- 入口 A / 入口 B 主链路回归
- 局部 lint 和 typecheck

---

## Example 3: Backend-style answer

I would keep the route handlers thin and extract the business flow into one shared service.

### Options
1. Patch both handlers independently  
   Lower immediate cost, but duplication remains.

2. Extract one service/use-case and keep handlers as adapters  
   Best balance of cleanliness and low risk.

3. Redesign handlers, repositories, and domain structure together  
   Too broad for the requested change.

### Recommendation
Use option 2.

### Design
- Keep parsing/auth/response shaping in handlers
- Move orchestration and decision-making into one service
- Isolate third-party I/O behind existing adapters or a small new boundary

### Flow
1. Handler validates input
2. Handler calls the service
3. Service runs the business flow
4. Service returns a stable result
5. Handler maps it back to HTTP/CLI/job output

### Change scope
- Add one service
- Move shared logic into it
- Update current handlers/jobs to delegate
- Keep external behavior unchanged

### Validation
- Unit-test the service decision path where practical
- Run targeted integration/regression on the affected endpoint/job
- Run focused static checks

---

## Example 4: Language for rejecting overdesign

Useful lines:

- “I wouldn’t split this further yet; the extra abstraction doesn’t pay for itself in this change.”
- “This solves the ownership problem without turning the feature into a framework.”
- “The cleaner-looking option is broader than the problem we’re actually solving.”
- “For now, one shared boundary is enough; if reuse expands later, we can split again from there.”
