# MAPS-0：Mod Card

> 规范版本：0.1（Draft）· 名片 `schema`：1
> 关键词 MUST / SHOULD / MAY 按 RFC 2119 理解。

## 1. 范围

Mod Card 是模组对 Agent 的**入口文档**。它回答"这是什么、给世界加了什么、从哪进、边界在哪"，不替代加载器元数据，也不复制配方与状态。

名片是**静态**的：宿主只读 JSON，不执行模组代码、不发起网络请求。

## 2. 文件

模组 MUST 在 jar 内提供以下任一位置的名片：

1. `agent/modcard.json`（首选）；
2. `assets/<modid>/agent/modcard.json`（资源目录受限时的回退）。

约束：

- MUST 是 UTF-8 编码的 JSON，不带 BOM；
- MUST NOT 包含可执行代码、脚本、二进制载荷；
- MUST NOT 包含密钥、令牌、玩家数据或任何隐私信息；
- SHOULD 小于 256 KB；
- MAY 在构建时生成，但生成结果必须随 jar 发布。

## 3. 顶层字段

### 3.1 必需

| 字段 | 类型 | 说明 |
|---|---|---|
| `schema` | integer | 固定为 `1`。不兼容变更必须递增。 |
| `id` | string | 模组 id，`^[a-z0-9_]+$`，MUST 与加载器元数据一致。 |
| `name` | string | 显示名。 |
| `version` | string | 模组版本，MUST 与加载器元数据一致。 |
| `environment` | string | `client` / `server` / `both`。 |
| `minecraft` | string | 支持的 Minecraft 版本或范围，如 `"1.21.1"`、`"[1.21,1.22)"`。 |
| `loaders` | string[] | 至少一项；已知值 `neoforge`、`forge`、`fabric`、`quilt`；允许其他。 |
| `summary` | string | 给人看的一句话，MUST ≤ 200 字符。 |
| `agent` | object | 见 §3.2。 |

### 3.2 `agent` 对象

必需：

| 字段 | 类型 | 说明 |
|---|---|---|
| `one_liner` | string | 给 Agent 看的一句话：它给世界加了什么，MUST ≤ 200 字符。可以与人读的 `summary` 不同。 |
| `entrypoints` | string[] | 入口列表；没有代码级入口时 MUST 是空数组，不能省略。 |

推荐（SHOULD）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `domains` | string[] | 领域标签：`kinetics`、`food`、`farming`、`logistics`…… |
| `verbs` | string[] | 动作入口，如 `create.machine.start`。 |
| `state` | string[] | 状态入口，如 `scan.block.details.create`。 |
| `knowledge` | string[] | 知识入口：`recipes`、`uses`、`info`、`machine_map`、`tutorial`、`skill`；允许其他。 |
| `good_at` | string[] | 擅长什么，帮助 Agent 选型。 |
| `not_good_at` | string[] | 不擅长什么，防止过度期待。 |
| `requires_client` | boolean | 是否只在主人客户端可用（JEI、Ponder 这类）。缺省 `false`。 |
| `permissions` | string[] | 归属/领地/危险动作：`claims`、`owner`、`consent`、`destructive`…… |
| `docs` | string[] | 深入阅读的文件或 URL。 |
| `examples` | string[] | 典型任务的一句话示例。 |
| `limitations` | string[] | 已知限制：性能、未覆盖的机器、版本差异。 |

### 3.3 其他可选字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `authors` | string[] | 作者/维护者。 |
| `license` | string | SPDX 标识或许可证名。 |
| `description` | string | 较长的介绍；仍应简短。 |
| `requires` | string[] | 硬依赖的模组 id。 |
| `optional` | string[] | 可选联动的模组 id。 |
| `content` | object | 内容索引：`blocks`、`items`、`machines`、`entities`、`dimensions`、`tutorials`；值为数量或字符串数组。 |
| `links` | object | 字符串到 URL：`modrinth`、`curseforge`、`source`、`issues`、`docs`。 |
| `maps` | object | 自报的符合级别：`{ "level": 0-3, "spec": "0.1" }`；见第 7 节。 |

## 4. 命名与语义约定

- **id**：小写字母、数字、下划线；与模组 id 一致。
- **版本**：字符串，建议语义化版本；不要求可解析。
- **entrypoints**：MAPS 把它们当作**不透明字符串**。宿主定义语法；推荐 `<host>.<kind>:<name>`，例如：
  - `buddy.api:create`（Buddy 的 API 组）
  - `buddy.details:create`（`scan.block.details.create`）
  - `knowledge:recipes`
  - `skill:docs/farming.md`
  - `doc:docs/machine-map.md`
  宿主 MUST 忽略不认识的入口，而不是报错。
- **state**：推荐 `scan.block.details.<id>`、`scan.entity.details.<id>` 这类路径；具体由宿主解释。
- **knowledge**：`recipes`、`uses`、`info` 是通用动词；`machine_map`、`tutorial`、`skill` 是推荐扩展。
- **requires_client**：为 `true` 时，宿主 SHOULD 在主人客户端不在线时把相关入口标为不可用，而不是让调用超时。
- **permissions**：名片只**声明**，不执行；真正的裁决在运行时。
- **content**：只做索引，不复制配方/状态；数量可以是近似值，但 SHOULD 准确。

## 5. 发现与校正

### 5.1 宿主扫描

宿主 SHOULD：

1. 只扫描已加载模组的 jar，不执行模组代码；
2. 先找 `agent/modcard.json`，再找 `assets/<modid>/agent/modcard.json`；
3. 用加载器元数据校正 `id` / `version`；不一致时记诊断，以加载器元数据为准；
4. 把名片合并成"已装模组目录"，供 Agent 选型；
5. 忽略未知字段，保留给未来版本。

### 5.2 运行时校正

名片是**承诺**，运行时注册表是**事实**。宿主 SHOULD 把以下内容对一遍：

- `agent.verbs` ↔ 实际注册的 API 组/函数；
- `agent.state` ↔ 实际注册的状态回答者；
- `agent.knowledge` ↔ 实际注册的知识来源；
- `agent.requires_client` ↔ 客户端能力是否上报。

缺失的入口记诊断；不要静默假装可用。

## 6. 安全

- 名片 MUST NOT 包含密钥、令牌、玩家数据、服务器地址等隐私信息。
- 名片 MUST NOT 包含可执行内容或要求宿主执行的内容。
- `links` SHOULD 使用 `https`；宿主打开链接前应提示或过滤。
- `permissions` 只是声明；宿主 MUST 在运行时按自己的权限层裁决，不能因为名片写了 `claims` 就放行。

## 7. 符合级别

| 级别 | 要求 |
|---|---|
| **MAPS-0** | 本规范全部 MUST 满足；名片可通过 schema 校验。 |
| **MAPS-1** | 在 MAPS-0 之上，提供至少一种知识入口（配方/用途/说明/机器映射）或一份结构化技能。 |
| **MAPS-2** | 在 MAPS-1 之上，提供状态快照 + 语义动作 + 类型化失败码。 |
| **MAPS-3** | 在 MAPS-2 之上，提供试算/干跑、进度事件、权限与审计。 |

`agent.entrypoints` 为空数组是合法的 MAPS-0：模组可以只有名片，没有代码级集成。

自报级别写在 `maps.level`；宿主 SHOULD 验证，而不是直接相信。

## 8. 校验清单

- [ ] 文件位于 `agent/modcard.json` 或 `assets/<modid>/agent/modcard.json`
- [ ] UTF-8、无 BOM、无代码/密钥/玩家数据
- [ ] `schema` = 1
- [ ] 所有必需字段存在且类型正确
- [ ] `id` / `version` 与加载器元数据一致
- [ ] `agent.entrypoints` 存在（可为空数组）
- [ ] 入口字符串不会被宿主误解为可执行代码
- [ ] `requires_client` 与实际情况一致
- [ ] 通过 [`../schema/modcard.schema.json`](../schema/modcard.schema.json) 校验

## 9. 示例

- [`../examples/create.modcard.json`](../examples/create.modcard.json)
- [`../examples/farmers-delight.modcard.json`](../examples/farmers-delight.modcard.json)

示例中的 `entrypoints` 可以描述**提议中的**集成；在实现落地前，宿主会在运行时校正阶段把它们标为缺失。
