# 让 Agent 玩得动模组：模组侧接口面设计笔记

> 以《农夫乐事》（Farmer's Delight）与《机械动力》（Create）为例，讨论一个模组需要暴露什么，外部大脑才能在 Buddy 的身体上"看懂—操作—排错—复用"。
> 调研日期：2026-11-27。对应 Buddy 代码：`Buddy-MC/Buddy-mod`（Buddy 0.1.0 / Minecraft 1.21.1 / NeoForge 21.1.233）。
> 本文是 **MAPS** 规范的设计笔记与背景说明；规范性条款见 [`../spec/mod-card.md`](../spec/mod-card.md) 与 [`../spec/playability.md`](../spec/playability.md)，名片 schema 见 [`../schema/modcard.schema.json`](../schema/modcard.schema.json)。

## 0. 结论

Agent 玩模组，不是"会点 GUI"，而是五件事：

0. **先知道这是什么**（mod card）：身份、版本、环境、内容、入口、边界；
1. **知道怎么做**（knowledge）：配方、机器、工序、前置条件；
2. **看见现在是什么状态**（state）：机器里有什么、进行到哪、为什么停着；
3. **能做**（actions）：每个界面操作都有对应的语义动作，且不依赖像素坐标；
4. **知道为什么不行、做完会怎样**（feedback & constraints）：失败码、进度、成本、权限。

拿这两个模组对照：

- **农夫乐事**的缺口是"小而隐"。配方是标准 JSON，物品栏也有 NeoForge capability 可读；但锅/砧板/煎锅的**热度、进度、槽位角色、匹配到的配方**都藏在 BlockEntity 和 GUI 里。
- **机械动力**的缺口是"大而隐"。配方是结构化的，但**机器怎么搭、动力网络里 SU/RPM 多少、传送带/列车/Contraption 在干什么**既不在配方里，也不在方块状态里。

所以第一件该补的不是 UI，而是**模组名片（Mod Card）**：一张随 jar 发布、不执行模组代码就能读的 `agent/modcard.json`，回答"这是什么、给世界加了什么、Agent 从哪进、边界在哪"。有了它，Agent 才能在几十个模组里决定先看谁、该信谁。

所以最值钱的不是再加一个给玩家看的 UI，而是**机器可读的接口面**：

```
status(target) -> state          // 状态快照，稳定字段名
actions(target) -> verbs         // 这台机器支持哪些语义动作
do(target, verb, args) -> result // 校验、执行、失败码；长活返回任务
knowledge(machine, item) -> graph // 配方 + 机器映射 + 工序顺序
events -> stream                 // 开始/进度/完成/失败/卡住
```

UI 继续给人看；上面这五样是给 Agent 看的。两者可以共享同一份底层状态。

Buddy 已经提供了通用的挂载点，模组**不需要改 Buddy** 就能挂上去：`buddy.api(...)`、`Questions.BLOCK/ITEM/ENTITY/CLICK/PERMIT`、`knowledgeSource(...)`、`Stations`、`registerForClientMod`。本文最后给出具体对接方式。

---

## 1. 模组名片（Mod Card）

### 1.1 为什么先要一张名片

现在的模组对 Agent 的第一手信息来自三处：

- **加载器元数据**（`neoforge.mods.toml` / `fabric.mod.json`）：id、版本、依赖、许可证；
- **平台页面**（Modrinth / CurseForge）：给人看的介绍、截图、更新日志；
- **游戏内表现**：方块、物品、配方、GUI。

这三处都不适合"先做判断"：加载器元数据只有机器字段；平台页面是营销文案，而且不在玩家机器上；游戏内表现要玩到才知道。Agent 面对一个装了上百个模组的整合包，第一步不是读配方，而是**决定先看哪个模组**。名片就是这第一步。

名片回答五个问题：

1. 这是什么（id、名字、版本、作者、许可证、环境、依赖）；
2. 它给世界加了什么（一句话 + 内容索引：方块/物品/机器/生物/维度）；
3. 它给 Agent 留了什么入口（API 组、block details、knowledge source、技能/文档）；
4. 它不擅长什么（限制、客户端依赖、性能、权限）；
5. 想深入时去哪儿读（文档、示例、Ponder、skills）。

### 1.2 三层信息，别混在一起

| 层 | 谁读 | 位置 | 特点 |
|---|---|---|---|
| 加载器元数据 | 加载器 | `META-INF/neoforge.mods.toml` / `fabric.mod.json` | 必需、最小、机器字段 |
| **模组名片** | Agent / 人 / 工具 | jar 内 `agent/modcard.json`（或 `assets/<modid>/agent/modcard.json`） | 静态、可读、版本化、不执行代码 |
| 运行时能力 | Buddy / Agent | 注册表（API 组、`Questions.*` 回答者、`ClientMods.provided`） | 真实可用性、随环境变化 |

名片是**承诺**，运行时能力是**事实**。两者要能对上：名片说有的入口，运行时没注册就是 bug；运行时多出来的入口，名片要补。Buddy 启动时可以扫一遍已加载模组的 jar，把名片汇总成"已装模组目录"，再用运行时注册表校正。

### 1.3 名片至少写什么

最小版（5 个字段）：

- `id` / `name` / `version`
- `environment`（client/server/both）、`minecraft`、`loaders`
- `summary`：一句话，给人看
- `agent.one_liner`：一句话，给 Agent 看（它和 `summary` 可以不同）
- `agent.entrypoints`：API 组 / details 名 / knowledge source / skill 路径；没有就写 `[]`

完整版建议：

| 字段 | 内容 |
|---|---|
| `id`, `name`, `version`, `authors`, `license` | 身份与归属 |
| `environment`, `minecraft`, `loaders`, `requires`, `optional` | 环境与依赖 |
| `summary`, `description` | 人读的介绍（短、具体） |
| `agent.one_liner`, `agent.domains` | Agent 读的一句话 + 领域标签 |
| `agent.verbs` | 动作入口，例如 `create.machine.start` |
| `agent.state` | 状态入口，例如 `scan.block.details.create` |
| `agent.knowledge` | 知识入口：`recipes`、`uses`、`info`、`machine_map`、`tutorial` |
| `agent.good_at` / `agent.not_good_at` | 能力边界，防止 Agent 过度期待 |
| `agent.requires_client` | 是否只在主人客户端可用（JEI/Ponder） |
| `agent.permissions` | 归属/领地/危险动作 |
| `agent.docs` / `agent.examples` | 深入阅读与示例 |
| `agent.limitations` | 已知限制（性能、未覆盖的机器、版本差异） |
| `content` | 方块/物品/机器/生物/维度索引（可选，便于规划） |
| `links` | Modrinth/源码/问题跟踪 |

### 1.4 一个例子

```json
{
  "schema": 1,
  "id": "create",
  "name": "Create",
  "version": "6.0.10+mc1.21.1",
  "authors": ["Simibubi and contributors"],
  "license": "MIT",
  "environment": "both",
  "minecraft": "1.21.1",
  "loaders": ["neoforge"],
  "requires": [],
  "optional": ["jei"],
  "summary": "Kinetic power, processing machines, contraptions and trains.",
  "agent": {
    "one_liner": "Build kinetic power networks and processing lines, then let them run.",
    "domains": ["kinetics", "processing", "logistics", "trains", "building"],
    "entrypoints": ["buddy.api:create", "Questions.BLOCK:create"],
    "verbs": ["create.kinetics.status", "create.machine.start", "create.contraption.assemble"],
    "state": ["scan.block.details.create.kinetics", "scan.block.details.create.machine"],
    "knowledge": ["recipes", "machine_map", "ponder_steps"],
    "good_at": ["turning SU/RPM into machine chains", "moving structures"],
    "not_good_at": ["planning a whole line without SU/space checks", "reading a network from block state alone"],
    "requires_client": false,
    "permissions": ["claims", "contraption_owner", "train_owner"],
    "docs": ["docs/machine-map.md", "docs/ore-line.md"],
    "examples": ["create a 64 RPM mixer line from a water wheel"],
    "limitations": ["no built-in dry-run planner", "trains need loaded chunks"]
  },
  "content": {
    "blocks": 200,
    "items": 150,
    "machines": ["mechanical_press", "mechanical_mixer", "encased_fan"],
    "tutorials": ["ponder:create/mechanical_press"]
  },
  "links": {
    "modrinth": "https://modrinth.com/mod/create",
    "source": "https://github.com/Creators-of-Create/Create"
  }
}
```

### 1.5 放哪里、怎么被读到

- **放 jar 里**：`agent/modcard.json`（或 `assets/<modid>/agent/modcard.json`）。静态 JSON，不执行模组代码；Buddy 可以只读 jar 就列出来。
- **不要只放 README**：README 是给人看的，长度、结构、版本都不稳定；名片是给机器读的。
- **不要只依赖 Modrinth**：玩家可能离线、整合包可能改过、版本可能不同；名片随 jar 走。
- **Buddy 侧（提议）**：启动时扫已加载模组的名片，合并成 `installed_mods`；`buddy_guide` / 系统提示里放一份精简目录；需要细节时再 `buddy.mod.card("create")`。MSP 的 manifest 也可以带上"已装模组"的摘要。
- **运行时校正**：把名片里的 `verbs/state/knowledge` 与真实注册的 API 组、`Questions.*` 回答者、`ClientMods.provided` 对一遍，缺失的记进日志/诊断页。

### 1.6 名片与技能的分工

- **名片**：是什么、能做什么、入口在哪、边界在哪。短、稳定、静态。
- **技能（SKILL.md）**：怎么完成一类任务，步骤、示例、坑。可以长，可以随版本改。
- **Ponder/教程**：人看的；如果能导出成结构化步骤，就可以作为技能的一部分。
- **配方知识**：`recipes/uses/info` 的来源；名片只写"有配方知识"，不复制配方。

一个模组可以只有名片（0 级集成），也可以名片 + 技能（1 级），再往上才是代码级状态/动作。

## 2. Buddy 现在能给模组什么，给不了什么

### 2.1 身体与通用动词

Buddy 的同伴是服务端假玩家（`BuddyPlayer extends ServerPlayer`），所有操作都是"真的按键、真的打开界面"，不是模拟数据。当前与被模组最相关的几组：

| 组 | 能力 | 对模组的限制 |
|---|---|---|
| `buddy.use.*` | 右键一格/一只实体、空手右键、左键一下、上床 | 返回 `accepted`，**不返回世界变成了什么** |
| `buddy.scan.block` | 方块 id/状态、硬度、工具、够不够得着、`details` | `details.storage` 只覆盖物品/流体/能量（NeoForge capability）；进度、热度、网络状态要靠回答者补 |
| `buddy.container.*` | 往"打开一个界面"的方块里放/取，Shift 点击式 | `menu = null`，不挑界面；**不能指定槽位角色**（原料/燃料/容器/产物），也不能读进度 |
| `buddy.furnace.*` / `buddy.brewing.*` / `buddy.anvil.*` … | 原版工作站，按槽位角色操作 | 每座工作站绑定了具体的 `AbstractContainerMenu` 类型；模组自己的 GUI 机器没有对应组 |
| `buddy.inv.*` | 背包、合成、吃、书、丢东西 | 只认原版合成与世界配方表 |
| `buddy.inv.recipes/uses/info` | 配方/用途/说明，原版 + 模组客户端上的 JEI | 配方是**只读知识**；没有机器怎么搭、怎么开、为什么停 |

其中 `buddy.scan.block` 的 `details` 是插件扩展点：回答者按名字把自己的字段挂进去（`details.storage`、`details.furnace` 就是这么来的）。这是模组暴露状态最省事的一扇门。

### 2.2 扩展点

`BuddyApi`（`buddy-api/common/src/main/java/com/buddymc/buddy/api/BuddyApi.java`）是扩展 Buddy 的唯一入口：

| 方法 | 用途 |
|---|---|
| `api(group, summary, Class)` | 登记一组 Lua 可见的动词，如 `mymod.machine.start(...)`；`@Fn/@Doc/@Example/@SeeAlso` 直接生成给模型的 API 索引和帮助 |
| `answer(Questions.BLOCK, name, describer)` | `buddy.scan.block(...).details.<name>`：机器状态、热度、进度、网络 |
| `answer(Questions.ITEM, name, describer)` | 物品的额外数据：耐久、内容物、模组自定义组件 |
| `answer(Questions.ENTITY, name, describer)` | 实体状态：模组生物、载具、NPC |
| `answer(Questions.CLICK, name, answerer)` | 右键**会对世界做什么**，供权限层在按下之前裁决（放置机器的物品尤其需要） |
| `answer(Questions.PERMIT / BLOCK_OWNER / ENTITY_OWNER, ...)` | 权限与归属：领地、机器主人、载具主人 |
| `knowledgeSource(kind, ...)` | 往 `recipes` / `uses` / `info` 三个动词里加来源 |
| `taskRunner(record, runner)` | 登记一种长活；函数返回 `Job.of(...)` 后按刻执行 |
| `bundleContent(root)` | 随 jar 带技能（SKILL.md）、Lua 模块、人设、模型档案 |
| `on(CompanionEvent...)` | 订阅同伴生命周期事件 |
| `registerForClientMod(namespace, modId, plugin)` | 目标模组只在客户端时的联动（JEI/Litematica 模式） |

另外，函数签名里的 `ServerCall` / `ClientCall` 决定了在哪执行：`ServerCall` 在服务端主线程，`ClientCall` 投回主人客户端执行（适合 JEI、界面、玩家本机数据）。`@Fn` 的说明是**给 LLM 的文档**，写得像对同事解释操作，而不是像 Javadoc。

### 2.3 现在的硬缺口

- `buddy.use.block` 只回 `accepted`，不回"放了什么、点燃了什么、开了什么界面、失败在哪一步"。
- 没有通用的"机器状态"问题。`details.storage` 只有物品/流体/能量；热度、RPM、SU、配方进度、模式、过滤器都不覆盖。
- `Stations` 能打开任意界面做 Shift 点击式搬运，但**按角色指定槽位**和**菜单专属动作**（选配方、改模式、按按钮）需要按菜单逐类集成。
- 知识是"配方中心"的：没有机器结构、工序顺序、速度/热量要求、失败模式；JEI 还会按 40 条配方、每侧 12 项截断。
- 没有发现机制：Agent 只知道注册了哪些 API 组；方块/物品本身不声明"我能做什么"。
- 没有"试算/干跑"：不能先问"这样搭要多少 SU、要几秒、缺什么"，只能做完再看。

---

## 3. 案例一：农夫乐事

### 3.1 玩法结构

一条典型链路是：**种/收 → 砧板切 → 锅/煎锅/灶台做 → 装盘/盛宴 → 吃出效果**。

从 1.21.1 NeoForge 的 jar 里能直接读到：

- 配方类型：`farmersdelight:cooking`、`farmersdelight:cutting`（另有 `create`、`immersiveengineering`、`silentgear` 等兼容配方）。
- `cooking` 配方：`ingredients`、`result`、`experience`、`recipe_book_tab`。
- `cutting` 配方：`ingredients`、`result`（可带概率，`ChanceResult`）、`tool`（`item_ability` + 物品/标签）、`sound`。
- `CookingPotBlockEntity` 实现 `MenuProvider`、`HeatableBlockEntity`、`RecipeCraftingHolder`，有 `MEAL_DISPLAY_SLOT`、`CONTAINER_SLOT` 和物品处理器 `CookingPotItemHandler`。
- `CuttingBoardBlockEntity` 有 `ItemStackHandler` 和 `CachedCheck<CuttingBoardRecipeInput, CuttingBoardRecipe>`。
- `SkilletBlockEntity` 有 `cookingTime` 和 `HeatableBlockEntity`；`AbstractStoveBlockEntity` 有 `items` 与 `cookingProgress`。
- `HeatableBlockEntity` 提供 `isHeated(level, pos)` / `requiresDirectHeat()`。

### 3.2 Agent 会卡在哪

1. **锅里有哪几样、哪一格是容器、匹配到哪条配方、熟没熟**，看 `scan.block` 只能看到物品清单，看不到角色、进度、热度、配方。
2. 通用 `buddy.container.put` 能 Shift 点击把东西塞进锅，但**不知道塞的是食材还是容器**，也可能把碗塞进食材槽。
3. 砧板没有界面，靠右键；`buddy.use.block` 能按，但**不知道手里工具对不对、会切出什么、有没有概率产物**。
4. 煎锅/灶台有进度和热度，但 Agent 只能反复看方块，看不到 `cookingTime` / `cookingProgress`。
5. 配方链是断的：知道"苹果酒要 2 苹果 + 糖"，但不知道"我现在缺糖、锅没热、还要一个碗"。
6. 盛宴/装盘的份数、放置状态没有语义查询。

### 3.3 最小集成（不改农夫乐事，只加 Buddy 插件）

**状态**：给四个方块各注册一个 `Questions.BLOCK` 回答者。

```lua
-- buddy.scan.block(pot).details.farmersdelight
{
  kind = "cooking_pot",
  heated = true,
  requires_direct_heat = false,
  container = "minecraft:bowl",
  ingredients = { { item = "minecraft:apple", count = 2 }, { item = "minecraft:sugar", count = 1 } },
  meal = "farmersdelight:apple_cider",
  progress_percent = 42,
  cook_seconds_left = 12,
  recipe = "farmersdelight:cooking/apple_cider",  -- 匹配到的配方；没有就是 nil
  blocked = nil                                   -- "no_heat" | "no_recipe" | "container_full" | "not_heated"
}
```

```lua
-- buddy.scan.block(board).details.farmersdelight
{
  kind = "cutting_board",
  input = "minecraft:acacia_log",
  tool = "minecraft:iron_axe",
  results = { { item = "minecraft:stripped_acacia_log", count = 1, chance = 1.0 },
              { item = "farmersdelight:tree_bark", count = 1, chance = 1.0 } },
  recipe = "farmersdelight:cutting/acacia_log",
  blocked = nil                                   -- "wrong_tool" | "no_recipe"
}
```

煎锅/灶台同理：`progress_percent`、`cook_seconds_left`、`heated`、`items`。

**动作**：注册一组 `buddy.api("farmersdelight", ...)`：

```lua
farmersdelight.pot.put({ cell = pot, role = "ingredient", item = "minecraft:apple", count = 2 })
farmersdelight.pot.put({ cell = pot, role = "container", item = "minecraft:bowl", count = 1 })
farmersdelight.pot.take({ cell = pot, role = "meal" })
farmersdelight.pot.status({ cell = pot })          -- 同 details，但可直接问

farmersdelight.board.cut({ cell = board, tool = "minecraft:iron_axe" })
farmersdelight.board.take({ cell = board, what = "result" })

farmersdelight.skillet.put({ cell = skillet, item = "minecraft:beef" })
farmersdelight.skillet.take({ cell = skillet })
```

实现上：锅用 `Stations.at(call, pot, CookingPotMenu.class, "cooking_pot", steps, ...)` 按 `CONTAINER_SLOT` / `MEAL_DISPLAY_SLOT` 和食材槽做 `Move`；砧板/煎锅/灶台用方块实体暴露的 `IItemHandler` + 右键，或直接走菜单。所有写操作经 `ServerCall` 的权限原语，不要绕过。

**知识**：两个来源。

- 如果装了 JEI：`farmersdelight:cooking` / `:cutting` 会通过 JEI 来源进入 `buddy.inv.recipes` / `uses`，但**机器映射**仍要补；
- 没装 JEI 或要补工序：注册 `knowledgeSource(KnowledgeKind.RECIPES, ...)` 和 `USES`，把锅/砧板/煎锅/灶台作为 `catalysts`，把 `tool`、`experience`、`chance` 写进 `alsoMakes` / `ingredients`。

**低成本补充**：`bundleContent` 带一份 SKILL.md，写清"锅要热、砧板要工具、碗是容器、盛宴怎么分"，比让模型自己悟快得多。

### 3.4 理想态（模组侧）

- 一份 `agent` 清单：锅/砧板/煎锅/灶台各自的状态 schema、动作、失败码、需要的工具/热量/容器。
- 失败码而不是"nothing happened"：`no_heat`、`not_heated`、`missing_container`、`wrong_tool`、`no_recipe`、`full`、`no_space`。
- 进度事件：`cooking_started`、`cooking_done`、`cut_done`、`food_burned`（如果适用）。
- 查询："我背包里这些能做什么"、"这条配方还缺什么"。

---

## 4. 案例二：机械动力

### 4.1 玩法结构

Create 的复杂度不在单条配方，而在**系统**：

- **动力网络**：动力源 → 轴/齿轮 → 机器；SU 容量与消耗、RPM、应力；调速器、仪表。
- **加工机器**：压床、搅拌盆 + 搅拌器、磨粉机、锯、部署器、注液器、风扇、打包机……配方类型 `create:mixing`、`pressing`、`crushing`、`milling`、`sawing`、`deploying`、`filling`、`emptying`、`haunting`、`compacting`、`sequenced_assembly` 等。
- **Contraption**：轴承/活塞/滑轮/龙门架把一组方块装成会动的结构；锚点、超级胶、移动、随动库存。
- **列车与物流**：轨道、车站、时刻表、包裹、传送带/漏斗/滑槽/过滤器。
- **图纸与教学**：蓝图台/蓝图炮；Ponder 场景。

jar 里能读到的结构化信息：

- 配方：`heat_requirement`、`ingredients`、`results`（可带 `chance`、流体 `amount`）、`loops`、`sequence`（`sequenced_assembly`）。
- 公共 API：`com.simibubi.create.api.*`，包括 `connectivity`、`contraption`、`behaviour`、`event`、`effect`、`data.recipe`、`stress`、`schematic`、`packager`、`registry`、`boiler`、`equipment`。
- 动力类：`KineticBlockEntity`、`GeneratingKineticBlockEntity`、`KineticNetwork`、`RotationPropagator`、`IRotate.StressImpact`、`BlockStressValues`、速度/应力仪表。
- Ponder：`CreateSceneBuilder` + 一批 instruction（客户端）。

### 4.2 Agent 会卡在哪

1. **配方不告诉你机器怎么搭**。`create:mixing` 只说要什么、出什么、要不要热；它不告诉你"搅拌盆 + 机械搅拌器 + 热源"，更不告诉你轴怎么连、要多少 RPM。
2. **网络状态不可见**。SU 容量/消耗、当前 RPM、是否过载、动力源在哪，都不在方块状态里；仪表是给人看的。
3. **Contraption 打破世界模型**。装起来之后方块坐标会动，库存/流体跟着走；Agent 不能再用"某格永久是某方块"来规划。
4. **工序顺序**。`sequenced_assembly` 的 `sequence` 是顺序的，`loops` 是重复次数；JEI 的平铺展示和 40 条截断会让 Agent 丢掉"先部署、再压、再锯"的过程。
5. **GUI/扳手/模式**。过滤器、模式、时刻表、装配、调速，很多是右键/界面操作，语义因方块而异。
6. **列车/物流是长状态**。时刻表、下一站、货物、包裹目的地，没有统一查询口。
7. **试算**。"这条产线要多少 SU、几个机器、多久出一件"目前只能搭完再看。

### 4.3 Create 已经有的资产

- 公开 API 覆盖了扩展点：`ConnectivityHandler`、`BlockMovementChecks`、`BlockEntityBehaviour`、`BlockStressValues`、`SchematicRequirementRegistries`、`SafeNbtWriterRegistry`、配方 datagen（`MixingRecipeGen` 等）。集成应优先用这些，而不是反射内部类。
- `BlockStressValues` 能给出应力影响/容量；`KineticNetwork` / `RotationPropagator` 能读网络；仪表方块本身就是"人读的网络状态"。
- Ponder 的场景构建器与 instruction 是结构化教学的雏形（虽然面向客户端动画）。

### 4.4 最小集成（不改机械动力，只加 Buddy 插件）

**状态**：用 `Questions.BLOCK` 给几类方块加 `details.create`。

```lua
-- 任意动力方块
{
  kinetics = {
    network = 12345,
    rpm = 64,
    source = { x = 100, y = 64, z = -20 },
    stress_impact = 8.0,
    overstressed = false
  }
}
```

```lua
-- 搅拌盆
{
  machine = {
    recipe_type = "create:mixing",
    recipe = "create:mixing/lava_from_cobble",  -- 匹配到的
    heat = "superheated",                        -- none | heated | superheated
    progress_percent = 60,
    inputs = { { item = "c:cobblestones", count = 1 } },
    outputs = { { fluid = "minecraft:lava", amount_mb = 50 } },
    speed_ok = true,
    blocked = nil                                -- "no_heat" | "no_stress" | "wrong_speed" | "no_space"
  }
}
```

```lua
-- Contraption 锚点
{
  contraption = {
    assembled = true,
    anchor = { x = 100, y = 64, z = -20 },
    blocks = 42,
    movement = "rotating",                       -- rotating | translating | stationary
    storage = { items = 3, fluids = 1 }
  }
}
```

列车、车站、传送带、漏斗、过滤器同理，字段用稳定的 snake_case。

**动作**：注册 `buddy.api("create", ...)`：

```lua
create.kinetics.status({ cell = shaft })
create.kinetics.connect({ from = shaft_a, to = shaft_b })
create.kinetics.set_speed({ cell = controller, rpm = 64 })

create.machine.status({ cell = basin })
create.machine.start({ cell = basin })
create.machine.stop({ cell = basin })
create.machine.set_filter({ cell = funnel, item = "minecraft:iron_ingot" })

create.contraption.assemble({ anchor = bearing, blocks = { ... } })
create.contraption.disassemble({ anchor = bearing })
create.contraption.status({ anchor = bearing })

create.train.schedule({ station = station, route = { "a", "b", "a" } })
create.train.status({ station = station })

create.schematic.place({ file = "line.nbt", at = pos, rotation = "clockwise_90" })
```

每个写动作先做校验：结构对不对、会不会过载、空间够不够、权限许不许；失败返回类型化错误。

**知识**：两件事最关键。

1. **机器映射**：`create:mixing → 搅拌盆 + 机械搅拌器 + 热源（heated/superheated）`；`sequenced_assembly → 按 sequence 的部署器/压床/锯 + 传送带`。这层映射今天只在 Ponder 和玩家脑子里。
2. **工序图**：`sequence`、`loops`、概率、流体量、时间；让 Agent 能画出"从矿到成品"的树，而不是只看到一条条平铺配方。

**试算**：`create.plan({ item = "create:precision_mechanism", rate_per_minute = 10 })` 返回机器链、需要的 SU、空间、材料、时间、瓶颈。这是 Create 最值得补的接口，因为它的失败大多来自"想得到、搭不对"。

### 4.5 理想态（模组侧）

- 统一的快照：`network`、`machine`、`contraption`、`train`、`logistics`。
- 类型化失败码：`no_stress`、`overstressed`、`wrong_speed`、`not_connected`、`no_heat`、`blocked`、`filter_mismatch`、`no_space`、`unloaded`。
- 事件：`machine_started`、`machine_done`、`overstressed`、`train_arrived`、`contraption_moved`。
- Ponder 的结构化导出：每一步的方块、位置、动作、说明，而不是只能看的动画。
- 权限：Contraption、列车、领地内的移动/放置要能被 `Questions.PERMIT` / `BLOCK_OWNER` 看见。

---

## 5. 通用接口面：给模组作者的清单

### 5.1 知识（Knowledge）

一条配方至少要说清：

| 字段 | 例子 | 为什么 |
|---|---|---|
| `type` | `create:mixing` | 机器/类别的入口 |
| `machine` / `catalysts` | 搅拌盆 + 搅拌器 | 配方不告诉你去哪做 |
| `inputs` | 物品/标签/数量/流体 | 替代品、标签、容器返还 |
| `outputs` | 数量/概率/副产物/流体 | `chance` 和 `alsoMakes` 不能丢 |
| `duration` | ticks / seconds | 规划与等待 |
| `heat` / `energy` / `speed` | `superheated`、SU、RPM | 前置条件 |
| `order` | `sequence`、`loops` | 工序不是集合 |
| `constraints` | 槽位、容器、工具 | 放错槽就是失败 |

另外要有**机器映射**：配方类型 → 需要的方块/结构/热量/速度；以及**结构化教学**（Ponder 的文本 + 步骤 + 方块），不是只有 UI 动画。

查询面建议：按产出、按输入、按机器、按背包（"我现在能做什么"）；名字用主人语言，id 稳定；带版本号。

### 5.2 状态（State）

- 字段名稳定、snake_case、英文；单位写进字段（`_mb`、`_ticks`、`_percent`、`_rpm`、`_su`）。
- 一格里放**角色**：`ingredient` / `container` / `fuel` / `input` / `output` / `byproduct` / `catalyst`。
- 进度：`progress_percent`、`seconds_left`、`eta`；状态：`idle` / `running` / `blocked` / `done`。
- 原因：`blocked = { code, detail, hint }`，不要只给一个 `false`。
- 内容：物品列表、流体罐（`fluid`、`amount_mb`、`capacity_mb`）、能量（`stored_fe`、`capacity_fe`）。
- 网络/结构：`network`、`members`、`stress_capacity`、`stress_used`、`rpm`；Contraption 的锚点/方块数/运动；列车的时刻表/下一站。
- 只读、便宜、可批量；不要每次只读一格就发一次网络往返（避免 N+1）；不要把原始 NBT 当唯一答案。

### 5.3 动作（Actions）

- **每个 GUI 操作都要有语义动词**：放入/取出按角色、按 ID 选配方、改模式、设过滤、启停、连接/断开、装配/拆解、定时/发车。
- 参数用物品 id/标签/数量、流体量、坐标/锚点，不用槽位像素。
- **先校验再执行**：能不能放、够不够、会不会过载、有没有权限；能返回 `dry_run` 预测更好。
- 幂等：重复调用安全；长活返回 `task`/`job`，带进度、取消、部分结果。
- 失败返回**类型化错误码 + 一句话提示**；不要"nothing happened"。

### 5.4 约束与反馈（Constraints & feedback）

- 前置：支撑面、朝向、净空、连通性、动力/热量/速度、结构完整、领地/归属。
- 成本：要消耗什么、要多久、会不会返还容器、会不会掉落。
- 事件：开始/进度/完成/失败/卡住/取消；网络变化、结构移动。
- 破坏性动作：先问，能撤销就撤销，不能就明确说。

### 5.5 发现与集成（Discovery）

- 一张随 jar 发布的**模组名片**（`agent/modcard.json`，见第 1 节）：身份、介绍、入口、边界、文档；不执行代码就能读。
- 一份可读的**能力清单**：版本、动作、状态 schema、失败码、权限、客户端/服务端侧。
- 优先用 Buddy 现成钩子；没有 Buddy 时也能被别的 Agent 读。
- 可选依赖：目标模组不在时零加载、零报错；不要硬绑。
- 客户端专属的能力（JEI、Ponder 动画）要标明"只在主人客户端"，服务端只读缓存/摘要。

### 5.6 安全与权限

- 归属：机器主人、领地、列车/Contraption 主人；`Questions.PERMIT`、`BLOCK_OWNER`、`ENTITY_OWNER` 是现成的入口。
- 破坏性/跨领地/移动方块的动作用同意层，不要绕过玩家的规则。
- 审计：谁在什么时候让哪台机器做了什么。

### 5.7 性能与并发

- 状态读取在服务端主线程、只读、短小；长活在异步任务里，按刻推进，可取消。
- 批量读、缓存、限制返回大小；避免在状态回答者里做慢查询。
- 确定性：同输入同结果；随机产物要能给出概率而不是"看运气"。

### 5.8 可评测性

- 场景夹具：一条完整链路（种植→切割→烹饪→装盘；采矿→粉碎→洗涤→熔炼→装配）。
- 状态快照/事件回放/失败注入；deterministic RNG；headless GameTest/e2e。
- 指标：成功率、步数、耗时、token、人工干预次数、错误恢复、计划质量、安全性、可复现性。

---

## 6. 与 Buddy 对接的具体做法

### 6.1 一个最小插件

```java
BuddyPlugins.register("mymod", buddy -> {
    // 1. 动词：她的程序里是 mymod.machine.start(...)
    buddy.api("machine", "My machines: status, start, stop, set_filter.", MachineApi.class);

    // 2. 状态：buddy.scan.block(pos).details.mymod
    buddy.answer(Questions.BLOCK, "mymod", (her, pos) -> {
        if (!(her.level().getBlockEntity(pos) instanceof MyMachine be)) return null;
        return Map.of(
            "state", be.stateName(),                 // idle/running/blocked
            "progress_percent", be.progressPercent(),
            "blocked", be.blockedCode()              // null 或 "no_power"
        );
    });

    // 3. 物品数据：背包清单与回执里的每一叠
    buddy.answer(Questions.ITEM, "mymod", (her, stack) -> ...);

    // 4. 右键会做什么：放置类物品必须登记，权限层才能在按下前裁决
    buddy.answer(Questions.CLICK, "mymod", (body, hit, stack) -> ...);

    // 5. 归属：我的机器/载具是谁的
    buddy.answer(Questions.BLOCK_OWNER, "mymod", (level, center, radius) -> ...);

    // 6. 配方知识
    buddy.knowledgeSource(KnowledgeKind.RECIPES, "what my machines make", List.of(), MyRecipes.class);

    // 7. 技能/模块随 jar 发布
    buddy.bundleContent(myJarBuddyRoot());

    // 8. 生命周期
    buddy.on(CompanionEvent.SPAWN, body -> ...);
});
```

函数类的写法：

```java
public final class MachineApi {
    public record Status(@Doc("The machine's cell.") BlockPos cell) {}

    @Fn("What this machine is doing now: state, progress, inputs, outputs, blocked reason.")
    @Example("mymod.machine.status({x = 100, y = 64, z = -20})")
    @Note("Read-only. blocked is a code plus a hint; progress_percent is 0-100.")
    public static Map<String, Object> status(ServerCall call, Status args) { ... }
}
```

`@Fn/@Doc/@Example/@Note` 不是装饰：它们生成 `buddy_guide` 里的 API 索引和每个函数的帮助，外部大脑就是靠这些文本决定怎么调用的。

### 6.2 客户端专属的联动

JEI 是现成范例（`plugins/jei`）：

- `BuddyJeiPlugin`（`@JeiPlugin`）拿 `IJeiRuntime`，进出世界时置空；
- `BuddyJei.install(host)` 由 `Builtin` 的 `Gate.openClientMod("jei", "jei", ...)` 在 JEI 在场时调用；
- 注册 `recipes` / `uses` / `info` 三个客户端知识来源；
- 服务端无条件登记签名（签名里没有 JEI 类型），客户端启动时把"我装了 JEI"报给服务端（`ClientCapsPayload` → `Presence`）；
- 调用时走 `ClientCall`，在主人客户端主线程查 `IJeiRuntime`，结果封顶（40 条配方、每侧 12 项、10 条说明 × 1500 字）；
- 加载器差异（流体数量）由 `JeiHost` 吸收。

一个模组可以照这个模式做"只在客户端才有的知识"（Ponder、JEI、EMI、地图 mod），但**权威状态和写操作必须在服务端**。

### 6.3 随包发布 vs 第三方插件

- Buddy 自带联动（ysm/tlm/kaleidoscope/curios/ftbquests/kubejs/jei）是 `compileOnly` 的源码模块，类被**平铺进发行 jar**，由 `Builtin` 的闸门在目标模组在场时安装；没有独立 mod id。
- 第三方模组不必改 Buddy：在自己的 mod 里调 `BuddyPlugins.register("yourmod", ...)`，Buddy API 在运行期可见（Buddy 内嵌 Buddy API）。
- 也可以只发布一份 `bundleContent` 技能（SKILL.md），零代码地教 Agent 你的模组流程；这是成本最低的"0 级集成"。

### 6.4 名片怎么进 Buddy

> 这一节是**提议**；当前 Buddy 还没有扫描名片或 `buddy.mod.card`，实现时应按 [MAPS 规范](../spec/mod-card.md) 落地。

- 模组把 `agent/modcard.json` 放进 jar；Buddy 启动时扫已加载模组的 jar（`ModList` + `ModJar`）读出来，不执行模组代码。
- Buddy 应提供 `buddy.mod.cards()` / `buddy.mod.card("create")`（或并入 `buddy_guide` 的精简目录），外部大脑通过 MSP/Lua 读到"已装模组目录"。
- 名片是静态承诺，注册表是运行时事实；两者对不上时记诊断，而不是让 Agent 猜。
- 没有代码的模组也能有名片；第三方插件可以用 sidecar 名片给没有内置名片的模组补一张。

### 6.5 已知的当前限制

- `buddy.use.block` 只回 `accepted`，不回结果；需要模组补状态查询。
- `buddy.container` 能对任意界面做 Shift 点击式放/取（`menu = null`），但**不能按角色指定槽位**；按槽位需要 `Stations.at(..., MenuType.class, ...)` 的专用组。
- `scan.block` 的 `details.storage` 只读 NeoForge capability 的物品/流体/能量；进度/热度/网络要 `Questions.BLOCK` 回答者。
- JEI 知识是只读且有上限；复杂配方（`sequenced_assembly` 的 5 道工序）需要自己的知识来源或结构化导出。
- 开发客户端里，`plugins/jei` 的 output 目前没进 `mods { }`，所以 dev 里 JEI 联动可能报 `NoClassDefFoundError: JeiHost`；发行 jar 里类在包内，正常。要 dev 验证需在 `buddy/neoforge/build.gradle` 的 `mods` 块补 `sourceSet project(':plugins:jei').sourceSets.main`。

---

## 7. 评测：什么叫"更好玩"

不要只问"它能不能做出来"，要问"它做得多像一个会玩的人"。

| 维度 | 指标 | 例子 |
|---|---|---|
| 成功 | 任务完成率 | 从种子到一盘菜；从矿石到精密构件 |
| 效率 | 步数、耗时、token | 同样任务，少走多少弯路 |
| 自主 | 人工干预次数 | 不需要玩家补一句"锅要热" |
| 恢复 | 失败后重试成功率 | 过载、没热、工具错，能不能自己查出来 |
| 计划 | 是否先算再搭 | Create 的 SU/空间/顺序 |
| 安全 | 越权/破坏次数 | 不碰别人的领地、不拆别人的 Contraption |
| 可复现 | 同种子同结果 | 评测与回放 |

夹具建议：

- **FD 农场到餐桌**：种/收 → 切 → 煮 → 装盘 → 吃；中途故意缺碗、缺热源、工具错。
- **Create 产线**：矿石 → 粉碎 → 洗涤 → 熔炼 → 装配；限 SU、限空间、限 RPM；中途过载/堵料。
- **列车/物流**：车站时刻表、包裹目的地、过滤器。
- **混合场景**：FD 的食材通过 Create 的传送带运输，考验跨模组规划。

---

## 8. 反模式

- **GUI 专属**：只有界面能做的事没有语义 API。
- **像素/槽位坐标**：让 Agent 记"点第 3 行第 2 格"。
- **状态藏在动画里**：Ponder 很好看，但 Agent 读不到。
- **原始 NBT 当接口**：字段不稳定、单位不明、没人知道哪个键重要。
- **只有配方**：不告诉机器怎么搭、要多少动力、什么顺序。
- **全有或全无**：失败只说 "failed"，没有原因和下一步。
- **无归属**：机器/载具/领地没有主人，Agent 一动手就是 grief。
- **阻塞调用**：一次状态查询卡 10 秒；长活没有任务/取消。
- **N+1**：逐格问网络状态，每条一次往返。
- **客户端权威**：把可被修改的客户端答复当真。
- **硬依赖**：目标模组不在就崩。
- **单位混乱**：mB / FE / ticks / 秒 / 百分比混着写。

---

## 9. 路线图

0. **名片标准**：`agent/modcard.json` 的 schema、jar 扫描器、运行时校正；先让 Agent 知道装了哪些模组、各自是什么。
1. **通用机器适配层**：任意 `AbstractContainerMenu` 的命名槽位、状态回答者注册、类型化结果；把 `Stations` 从"每个原版工作站一组"推进到"每个模组机器一张 schema"。
2. **知识 schema 扩展**：机器映射、工序顺序、热量/速度/应力、结构化教学（Ponder 导出），并纳入 `buddy.inv.recipes`。
3. **计划与试算**：`plan(target, rate)`、`dry_run`、成本/时间/空间/SU 估算；事件流。
4. **社区清单**：`agent.json` 或等价的 Buddy API 组 + lint + 场景测试，让"Agent 友好"成为可检查的发布项。

---

## 10. 附：两个模组的缺口对照

| 维度 | 农夫乐事 | 机械动力 |
|---|---|---|
| 配方 JSON | 有（cooking/cutting） | 有（mixing/pressing/sequenced_assembly…） |
| 机器映射 | 无（锅/砧板/煎锅/灶台靠常识） | 无（搅拌盆+搅拌器、部署器产线靠 Ponder） |
| 状态 | 物品栏可读；热度/进度/槽位角色不可读 | 库存/流体可读；SU/RPM/进度/网络不可读 |
| 动作 | 右键可用；语义动作（cut/put role）无 | 右键/扳手可用；语义动作（connect/assemble/set_filter）无 |
| 失败码 | 无 | 无 |
| 事件 | 无 | 无 |
| 权限 | 物品/方块归属通用；机器主人无 | Contraption/列车/领地归属无 |
| 结构化教学 | 无 | Ponder 有，但只给客户端动画 |
| 规划/试算 | 无 | 无（最痛） |
| 名片 | 无 | 无（可先由 sidecar 补） |
| 最小集成 | `Questions.BLOCK` + 一组 `farmersdelight.*` 动词 + 配方来源 | `Questions.BLOCK` + `create.*` 动词 + 机器映射/试算 |
| 理想集成 | 状态/动作/失败码/事件/技能包 | 网络/机器/Contraption/列车快照 + 计划 + Ponder 导出 |

---

## 11. 参考

- Farmer's Delight：<https://modrinth.com/mod/farmers-delight>
- Create：<https://modrinth.com/mod/create>
- 烟火凡人心（Flavor Immersed Daily）：<https://modrinth.com/mod/flavorimmerseddaily> / <https://www.mcmod.cn/class/15912.html>
- Buddy 扩展点：
  - `buddy-api/common/src/main/java/com/buddymc/buddy/api/BuddyApi.java`
  - `buddy-api/common/src/main/java/com/buddymc/buddy/api/Questions.java`
  - `buddy-api/common/src/main/java/com/buddymc/buddy/api/describe/Describer.java`
  - `buddy-api/common/src/main/java/com/buddymc/buddy/api/knowledge/Recipe.java`
  - `buddy/common/src/main/java/com/buddymc/buddy/tools/perception/ScanApi.java`
  - `buddy/common/src/main/java/com/buddymc/buddy/tools/station/ContainerApi.java`
  - `buddy/common/src/main/java/com/buddymc/buddy/tools/station/Stations.java`
  - `buddy/common/src/main/java/com/buddymc/buddy/tools/interact/UseApi.java`
  - `plugins/jei/src/main/java/com/buddymc/buddy/plugins/jei/`
- 模组名片：本文第 1 节（`agent/modcard.json`）
- Create API 包：`com.simibubi.create.api.*`（`connectivity`、`contraption`、`behaviour`、`stress`、`schematic`、`data.recipe` 等）
- 本文的 jar 观察基于 1.21.1 NeoForge 版本：`FarmersDelight-1.21.1-1.3.4.jar`、`create-1.21.1-6.0.10.jar`。
