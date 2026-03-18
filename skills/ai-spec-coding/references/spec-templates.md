# Spec Templates

按需复制和裁剪，不必每次全量使用。

## 1. Proposal 模板

```md
# Proposal

## Background
- 当前现状：
- 当前问题：
- 触发原因：

## Goal
- 目标：
- 业务价值：
- 成功标准：

## Non-goals
- 本次不做：

## Constraints
- 技术约束：
- 规范约束：
- 环境约束：

## Risks
- 风险点：
- 回滚点：
```

## 2. Design 模板

```md
# Design

## Overview
- 方案概述：

## Scope
- 涉及目录/模块：
- 不涉及内容：

## Data Flow
- 输入：
- 处理：
- 输出：

## API / Contract
- 受影响接口：
- 请求/响应变化：
- 枚举/状态变化：

## State / UI
- 页面状态：
- 空态/加载态/错误态：
- 权限与边界：

## Compatibility
- 向后兼容点：
- 迁移策略：

## Verification
- 类型检查：
- 自动化测试：
- 手工冒烟：
```

## 3. Spec 模板

```md
# Spec

## User-visible Behavior
1. 用户进入页面时
2. 用户执行操作时
3. 请求成功时
4. 请求失败时
5. 数据为空时
6. 权限不足时

## Interaction Rules
- 按钮启用/禁用条件：
- 表单校验规则：
- 列表筛选/分页规则：
- 弹窗/抽屉开关规则：

## Edge Cases
- 重复提交：
- 并发状态：
- 数据缺失：
- 跨页返回：
```

## 4. Tasks 模板

```md
# Tasks

## Task 1: 梳理现状
- Input:
- Action:
- Output:
- Acceptance:
- Verify:

## Task 2: 补充类型与服务层
- Input:
- Action:
- Output:
- Acceptance:
- Verify:

## Task 3: 实现页面/组件
- Input:
- Action:
- Output:
- Acceptance:
- Verify:

## Task 4: 联调与验证
- Input:
- Action:
- Output:
- Acceptance:
- Verify:
```

## 5. 重构任务模板

```md
# Refactor Plan

## Must Preserve
- 保持不变的行为：
- 保持兼容的接口：

## Problems Today
- 职责混杂：
- 目录混乱：
- 类型泄漏：
- 依赖方向错误：

## Refactor Steps
1. 只做目录/职责梳理，不改行为
2. 拆分公共与业务组件
3. 拆分 hooks / services / constants
4. 更新引用
5. 类型检查
6. 冒烟验证

## Exit Criteria
- 无行为回归
- 类型通过
- 关键页面可用
```
