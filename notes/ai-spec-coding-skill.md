# AI Spec Coding Skill

已在 OpenClaw workspace 中创建一个通用 AI 开发协作 skill，位置如下：

- `skills/ai-spec-coding/SKILL.md`
- `skills/ai-spec-coding/references/spec-templates.md`
- `skills/ai-spec-coding/references/patterns-and-outputs.md`
- `skills/ai-spec-coding/references/troubleshooting-playbook.md`
- `skills/ai-spec-coding/references/multi-agent-playbook.md`

已提交：

- commit: `297e1ce` — `Add ai-spec-coding skill`

## 核心设计

这个 skill 不是绑定 Claude Code，也不是只绑定前端，而是抽象成通用 AI 开发协作方法论：

- 先做任务分级：小改 / 标准化需求 / 复杂需求
- 对应三种模式：直接改 / 模板规则驱动 / Spec 驱动
- 三层规范体系：
  - 约束层
  - 示范层
  - 上下文层
- 三种常见失效模式：
  - 规范真空
  - 信息孤岛
  - 目标模糊
- 复杂排障采用实验驱动
- 大任务支持多 Agent 分工

## 目录结构

```text
skills/ai-spec-coding/
├── SKILL.md
└── references/
    ├── spec-templates.md
    ├── patterns-and-outputs.md
    ├── troubleshooting-playbook.md
    └── multi-agent-playbook.md
```

## 说明

这个 skill 是基于一篇关于 Claude Code / Spec Coding / SDD 实战经验的文章提炼出来的，但已经重写为一个更通用的 AI 开发 skill，可用于需求澄清、规范驱动开发、复杂重构、联调、排障和多 Agent 协作。
