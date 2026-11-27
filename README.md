# MAPS

## Mod Agent Playability Specification

### 模组智能体可玩性规范

> 状态：Draft 0.1 · 2026-11-27
> 维护者：Buddy-MC
> 参考实现：Buddy（[`../Buddy-mod`](../Buddy-mod)）
> 目的：让模组不只对人可玩，也对 Agent 可读、可操作、可排错、可规划。

## 一句话

**MAPS 是一份"模组对 Agent 可玩"的最低接口面约定。** 它不要求模组内置 AI，也不要求模组为某一个宿主写死；它要求模组把"是什么、能做什么、现在是什么状态、为什么不行"用机器可读的方式说出来。

## 问题

现在的模组对 Agent 的第一手信息来自三处：

- 加载器元数据（`neoforge.mods.toml` / `fabric.mod.json`）：只有 id、版本、依赖；
- 平台页面（Modrinth / CurseForge）：给人看的营销文案，且不在玩家机器上；
- 游戏内表现：要玩到才知道，状态多半藏在 GUI、BlockEntity、客户端动画里。

Agent 面对装了上百个模组的整合包，第一步不是读配方，而是决定先看哪个模组。然后才是"怎么做、现在怎么样、能不能做、做完会怎样"。

## 模型

```
MAPS-0  Card              这是什么？入口在哪？边界在哪？
MAPS-1  Knowledge         怎么做？机器怎么搭？工序是什么？
MAPS-2  State & Actions   现在是什么状态？能做哪些语义动作？失败码是什么？
MAPS-3  Feedback & Plan   进度、事件、试算、权限、审计
```

MAPS-0 是入口；MAPS-3 是理想态。一个模组可以只做 MAPS-0（一份 JSON），也可以逐级往上。

## 符合级别

| 级别 | 名称 | MUST | 典型成本 |
|---|---|---|---|
| **MAPS-0** | Card | jar 内 `agent/modcard.json`：身份、介绍、入口（可为空）、边界 | 一份 JSON |
| **MAPS-1** | Knowledge | 配方/用途/说明，或机器映射；或者一份结构化技能（SKILL.md） | 数据或文档 |
| **MAPS-2** | State & Actions | 状态快照 + 语义动作 + 类型化失败码 | 代码 |
| **MAPS-3** | Planning & Feedback | 试算/干跑、进度事件、权限与审计 | 代码 |

> 关键词 MUST / SHOULD / MAY 按 RFC 2119 理解。

## 仓库结构

```
MAPS/
├── README.md                     ← 本文件：定位、模型、级别
├── spec/
│   ├── mod-card.md               ← MAPS-0：Mod Card 规范
│   └── playability.md            ← MAPS-1/2/3：知识、状态、动作、反馈
├── schema/
│   └── modcard.schema.json       ← Mod Card 的 JSON Schema（schema = 1）
├── examples/
│   ├── create.modcard.json
│   └── farmers-delight.modcard.json
└── docs/
    └── agent-friendly-mods.md    ← 设计笔记（背景、案例、反模式）
```

> `examples/` 里的入口与动词可以是**提议中的**集成；规范允许名片先描述目标，宿主在运行时校正阶段会把未实现的入口标为缺失。

## 采用方式

### 模组作者

1. **MAPS-0**：在 jar 里放 `agent/modcard.json`（或 `assets/<modid>/agent/modcard.json`），按 [`spec/mod-card.md`](spec/mod-card.md) 填写；用 [`schema/modcard.schema.json`](schema/modcard.schema.json) 校验。
2. **MAPS-1**：如果没有代码 API，至少写一份结构化技能，说明这个模组的典型任务流程、配方与机器映射、常见失败。
3. **MAPS-2**：给机器/方块加状态快照和语义动词；所有 GUI 操作都要有等价动作；失败返回错误码而不是沉默。
4. **MAPS-3**：加试算/干跑、进度事件、权限与审计。

### 宿主 / Agent

1. 启动时扫描已加载模组的 jar，读取 `agent/modcard.json`，合并成"已装模组目录"；不要执行模组代码。
2. 把名片里的 `verbs/state/knowledge` 与运行时注册表对一遍；缺失的记诊断。
3. 先读 MAPS-0 决定看哪个模组，再按 MAPS-1/2 调用；MAPS-3 用于规划和失败恢复。

## 与 Buddy 的关系

- Buddy 是 MAPS 的**第一个参考实现**：`buddy.api(...)`、`Questions.BLOCK/ITEM/CLICK/PERMIT`、`knowledgeSource(...)`、`Stations`、`registerForClientMod` 在设计上对应 MAPS-0/1/2 的入口。
- 名片扫描与 `buddy.mod.card(...)` 目前是**提议**，尚未在 Buddy 中实现；设计笔记第 6.4 节描述了接入方式。
- MAPS 本身**不绑定 Buddy**：其他宿主可以定义自己的入口语法；名片里的 `entrypoints` 是宿主可解释的字符串。
- Buddy 侧的对接示例见 [`docs/agent-friendly-mods.md`](docs/agent-friendly-mods.md) 第 5–6 节。

## 版本

- 名片 `schema`：当前为 `1`；不兼容变更必须递增。
- 规范版本：`0.1`（Draft）。在 1.0 之前字段可以增删，但会记录变更。
- 兼容性原则：新增可选字段是兼容变更；删除/重命名必需字段是不兼容变更。

## License

- 规范文本（`README.md`、`spec/`、`docs/`）：**CC BY 4.0**，见 [`LICENSE-docs`](LICENSE-docs)。
- schema 与示例（`schema/`、`examples/`）：**MIT**，见 [`LICENSE-code`](LICENSE-code)。
- 引用时请注明 "MAPS — Mod Agent Playability Specification"（本仓库，Draft 0.1）与版权人。
- 版权人：Copyright (c) 2026 Buddy-MC。
