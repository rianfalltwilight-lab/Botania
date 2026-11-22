# 植物魔法 456 网络包与性能审计（大修第一阶段）

- 审计日期：2026-11-22
- 代码基准：`scex/456-20260822` 分支，即 `5d8262dc1`，与服务器使用的 `botania-neoforge-1.21.1-456-20260822.093314-4.jar` 同源
- 方式：只读源码审计，没有编译、没有运行，也没有服务器实测数据
- 范围：自定义网络包、方块实体同步、实体与物品同步、服务端 tick 的 CPU 开销

下文路径默认相对于 `Xplat/src/main/java/vazkii/botania/`，以 `NeoForge/` 开头的除外。

## 0. 结论摘要

最可能造成"网络包性能非常严重"的是三类问题。它们都会随设施规模或玩家数线性放大，而且全部是只为视觉效果发包，或者每 tick 改写物品数据：

1. **火花每传一次魔力就发一个粒子包**（N1）：连接对数 × 20 次/秒 × 附近玩家数。
2. **佩戴饰品每 tick 改写数据组件，触发 Curios 整组同步**（N2）：飞行中的芙格尔头冠、隐身斗篷、染血吊坠、用魔力戒指付费的各种饰品。每次改写都会把**完整 ItemStack** 发给佩戴者和所有能看到他的玩家，同时在服务端走一遍卸下再装备的流程。
3. **魔力脉冲衰减期间每 tick 同步实体数据**（N3）：每个正在衰减的脉冲每 tick 多发一个实体数据包，并且因为数据变脏，移动包也会每 tick 发送。

CPU 方面没有发现能单独拖垮服务器的问题，但有一批"空闲时也全范围扫描"的功能花和 O(N²) 的结构，见第 3 节。

**审计中推翻的一条**：子代理曾报告"发射器每 tick 重跑整条模拟路径"（`PositionProperties.fromEntity` 用了 `getBlockStateOn()`）。核对 1.21.1 原版源码后确认：`getBlockStateOn()` 只向下偏移 1e-5，并非 0.2（0.2 是 `getOnPosLegacy`）；飞行中的脉冲也没有支撑方块。所以只有采样点的 y 几乎正好落在整数上时才会误判，**不会**让发射器卡在每 tick 重算的状态。已降为 P2 写法问题（C-P2-1）。

### 严重程度定义

| 等级 | 含义 |
| --- | --- |
| P0 | 正常的大型基地玩法下就会刷屏发包或明显拖慢服务器，且随规模、玩家数放大 |
| P1 | 特定布局或玩法下明显 |
| P2 | 影响小，或只是顺手可改的问题 |

### 修复原则

除非特别注明，所有修复都必须**保持玩法完全一致**。只影响视觉的改动（例如粒子频率、客户端显示更新延迟）会单独标注。

## 1. 网络：P0

### N1 火花魔力流动粒子包：每对连接每 tick 一个包

- **位置**：`common/entity/ManaSparkEntity.java`
  - 168：扩散增强给玩家充能
  - 210：outgoing 循环
  - 240：inbound 循环
  - 290–298：发送函数，`sendToTracking(this, new SparkManaFlowEffectPacket(...))`
- **现象**：
  - 只为客户端画粒子，却每 tick 为每个传输对象单独发一个包。
  - inbound 循环在 `gained == 0` 时也照发（已核实第 236–240 行）。
  - 收包对象是火花追踪范围内的所有玩家（`clientTrackingRange(4)`，约 64 格）。
- **规模估算**：3 个支配火花各从 15 个池抽取，加 2 个隐性池各推送 10 个目标，共 65 对连接，约 1300 包/秒/人；附近 5 人就是约 6500 包/秒。
- **附带的 CPU 问题**：
  - 支配火花在池子没满时每 tick 都调用 `updateTransfers()`（173 行），即一次 25³ 的实体搜索、洗牌和过滤。
  - 池子在满与不满之间来回切换时，`notifyOthers` 会让范围内每个火花都重新搜索一遍（C-P1-6）。
- **修复**：
  - 先做：没有传输魔力时不发包。
  - 主方案：只在传输集合变化（开始或停止）时同步给客户端，由客户端每 tick 自己生成粒子，稳态接近 0 包/秒。
  - 退而求其次：每个火花每 tick 合并成一个包（列出所有对象），再加每 4 tick 发一次的节流。
  - 另外：支配火花的 `updateTransfers()` 改为在收到通知时，或每 10–20 tick 才重新计算。
- 以上均不改变玩法。

### N2 佩戴饰品每 tick 改写数据组件，触发 Curios 整组同步

- **机制**（已核实）：Curios 9.5.1 的 `CuriosEventHandler.tick` 每 tick 比较每个饰品槽的当前物品和上一 tick 的物品。只要 `ItemStack.matches` 不成立，就会：
  - 调用 `syncCurios`，用 `SPacketSyncStack` 把**完整 ItemStack** 发给佩戴者和所有追踪他的玩家；
  - 发出 `CurioChangeEvent`，并重算属性修饰符。
  - 原版装备栏也一样：数据组件一变，就会给附近玩家发 `ClientboundSetEquipmentPacket`。
- **每 tick 改写饰品或魔力物品的地方**：
  - `common/item/equipment/bauble/FlugelTiaraItem.java`
    - 230–271：生存模式飞行时 `REMAINING_TICKS` 每 tick 减 1。落地后每 2 tick（滑翔时每 6 tick）加 1，直到回满，默认要恢复约 2 分钟。
    - 100–111：飞行期间每 tick 调用 `requestManaExact` 扣魔力。
  - 所有每 tick 付魔力的饰品或工具都会改写付费的魔力物品。如果付费的是戴在 Curios 戒指槽里的魔力之戒，就触发上面那条整组同步：
    - `InvisibilityCloakItem.java:37`：每 tick 扣 2 点
    - `TaintedBloodPendantItem.java:107`
    - `RingOfChordataItem.java:54`：在水里时
    - 受损魔钢、元素工具或盔甲的自修复（`ManasteelArmorItem.java:60`，`ManasteelPickaxeItem.java:86` 等，每 tick 修 1 点耐久）
    - `LivingwoodBowItem.java:40`
  - `BandOfAuraItem.java:28`：每 50 tick（高级版每 10 tick）给戒指加 5 点魔力，戒指没满之前一直在同步。
- **规模估算**：一个飞行中的玩家，头冠 20 次/秒加付费戒指 20 次/秒，每次都发给自己和附近 4 人，就是约 200 个完整 ItemStack 包/秒。
- **修复**：
  - 头冠：飞行计时改存到服务端的玩家附件（它本来就是按玩家算的），只给本人发一个很小的 HUD 包；或者只在 HUD 的 10 格显示变化时、以及落地时写回物品。几个状态标志只在真正变化时写入。
  - 魔力扣费：给 `DefaultManaItemImpl` 加一个"待写入差值"缓存，读写魔力时都经过它，逻辑结果保持精确；每约 10 tick 写回一次数据组件，在物品被移动、丢出、玩家下线时立即写回。
  - 两者都不改变玩法。
- **前提**：服务器装了 Curios，魔力物品戴在戒指槽里。这个推断来自整合包常见配置，**需要确认服务器上 Curios 的实际版本**。

### N3 魔力脉冲衰减期间每 tick 同步实体数据

- **位置**：`common/entity/ManaBurstEntity.java`
  - 182–190：超过 `minManaLoss` tick 后每 tick 执行 `setMana(mana - loss)`
  - 678–680：`setMana` 写的是 `entityData.set(MANA, ...)`
- **现象**：
  - 原版只要实体数据变脏，就会每 tick 发 `SetEntityData`；而且移动包也会绕过 `updateInterval(10)`，同样每 tick 发送。
  - 普通发射器 60 tick 后开始衰减，射程约 12 格外，每个脉冲最多多发约 80 个包/人。盖亚发射器 120 tick 后开始衰减，射程约 48 格外。
  - 客户端只用 MANA 决定粒子大小和消散效果。
- **严重程度**：P1；远距离输送多的基地会接近 P0。
- **修复**：
  - 衰减改为服务端私有字段。
  - 客户端用已同步的 `START_MANA`、`MIN_MANA_LOSS`、`MANA_LOSS_PER_TICK` 加自己的 tick 计数推算当前魔力。
  - 只在非衰减的改动（`BoreLens:113`、`DamagingLens:40`、`TerraBladeItem:124`）时同步。
  - 不改变玩法。

## 2. 网络：P1

| 编号 | 位置 | 问题 | 修复（保持玩法） |
| --- | --- | --- | --- |
| N4 | `ManaPoolBlockEntity.java:394`，`EntityHelper.java:43-47` | 池里每个正在充能或放能的物品，每 tick 用 `syncItem`（先设为空再设回）重发整个 ItemStack，发给 6 个区块内所有追踪者。创造魔力平板放在一个持续不满的池里时会永远发下去 | 只在显示的填充格数变化、传输结束时，或每 20 tick 同步一次；物品没变化就不发。仅影响视觉 |
| N5 | `ManaMirrorItem.java:70-87` | 魔力镜每 tick 把绑定池的魔力写进物品，每 tick 一个背包槽位包；拿在手上时还会给附近玩家发装备包 | 读写都直接走绑定的池，物品上的显示值只在进度条像素变化时更新 |
| N6 | `TerraShattererItem.java:136-147` | 泰拉粉碎者开启时每 tick 扣自身魔力，拿在主手时每 tick 发装备包并重算属性 | 用 N2 的"待写入差值"方案 |
| N7 | `BotaniaEntities.java:30-35` | 脉冲追踪范围是 6 区块（原版投射物是 4），每个脉冲约 6–8 个包（生成、数据含镜片 ItemStack、移动、死亡、移除）。1000 魔力/tick 的普通发射器约 125 个脉冲/秒，约 900 包/秒/人 | 追踪范围降到 4（只影响远处能否看到）；`ping()` 缓存发射器引用 |
| N8 | `ManaSpreaderBlockEntity.java:259-268`，`NeoForge/.../NeoForgeCommonInitializer.java:596-599` | 用对象身份判断"接收方变了"。魔力虚空每次查询都 new 一个新对象，所以对着魔力虚空的发射器（转盘扫过时）每 tick 都立即全量同步。附属里返回新包装对象的 capability 也会这样 | 改为比较位置和维度 |
| N9 | `ManaSpreaderBlockEntity.java:352-360` | 更新包直接用 `saveAdditional` 写出全部存档 NBT（约 24 个键加镜片物品，约 600 字节，实际只需要约 30 字节），会放大 N8、转盘、首 tick 等每一次同步 | 只写客户端需要的字段，客户端单独读取 |
| N10 | `SpreaderTurntableBlockEntity.java:41-57` | 转盘上的发射器每 tick 重跑一次完整模拟（CPU），接收方在"有/无"之间切换时每次立即全量同步 | 只在发射器这一 tick 真的可能开火时才模拟；网络部分靠 N8 解决 |
| N11 | `ManaPoolBlockEntity.java:606-620` | 稀释魔力池容量 10000，1% 的阈值只有 100 魔力，基本每 13 tick 同步一次；包发给整个区块的追踪者，并不只是 64 格内 | 阈值加一个绝对下限（例如几千魔力）。仅影响视觉 |
| N12 | `RunicAltarBlockEntity.java:392-396` | 每收到一个脉冲就 `sendBlockUpdated`，不限流，魔力没变化也发；更新包含 16 格物品栏 | 只在值变化时同步并限流；越过 `manaToGet` 时立即同步（客户端要靠它播闪电效果） |
| N13 | `CorporeaSparkEntity.java:218`，`VanillaCorporeaNode.java:62` | 每抽取一个槽位发一次物品粒子，`ItemParticleOption` 带完整 ItemStack（潜影盒、书会很大），发给 32 格内玩家 | 每个火花每 tick 每种物品只发一次，粒子用去掉数据组件的物品。仅影响视觉 |
| N14 | `network/serverbound/DodgePacket.java:40-49` | 服务端先播放音效再校验戒指，并且不检查冷却。改过的客户端可以任意频率刷包，每个包都会向 16 格内所有玩家广播一次音效 | 先校验戒指和冷却，再播放音效 |
| N15 | `network/serverbound/IndexKeybindRequestPacket.java:38`，`IndexStringRequestPacket` | 请求不限速，按键请求的数量不设上限（字符串请求有 `MAX_REQUEST` 上限）；每个包都会触发整个网络扫描、系统消息、进度触发和生成掉落物 | 每个玩家加冷却，数量限制到 `min(最大堆叠, MAX_REQUEST)`，消息长度设上限 |
| N16 | `SpectatorItem.java:66-196` | 每 4 tick 扫描 25³ 方块实体加 49³ 实体，并把结果写进饰品的数据组件，结果一变就触发 N2 的 Curios 整组同步（列表可达数 KB） | 结果只发给佩戴者，单独发包，不存进物品 |

## 3. 服务端 CPU

### P1

| 编号 | 位置 | 问题 | 修复（保持玩法） |
| --- | --- | --- | --- |
| C-P1-1 | `flower/functional/BubbellBlockEntity.java:51-70` | 有魔力时每 tick 遍历半径 12 的整个立方体（约 7200 次 `getBlockState`），没有间隔限制 | 按区块段遍历，用调色板跳过不可能含水的段 |
| C-P1-2 | `flower/functional/ExoflameBlockEntity.java:41-60` | 绑定了池子就每 tick 做 605 次 `getBlockEntity`，每个熔炉都新建一个包装对象并查询配方 | 改为遍历相关区块的方块实体表，保持原来的迭代顺序 |
| C-P1-3 | `OrechidBlockEntity.java:73-90, 118-131`（含 Marimorphosis、Orechid Ignem） | 没东西可转换时没有冷却，每 2 tick 就重扫整个范围（Marimorphosis 3179 格） | 调色板跳过；缓存每个 BlockState 的匹配结果 |
| C-P1-4 | `flower/generating/MunchdewBlockEntity.java:112-130` | 附近没有树叶时每 4 tick 重扫 4913 格 | 调色板跳过 |
| C-P1-5 | `flower/functional/HopperhockBlockEntity.java:61-160` | 每 tick 搜索最大 21³ 范围；塞不进去的物品每 tick 对 6 个方向重试，每次都查 capability 并遍历所有槽位 | 每 tick 只解析一次各方向的物品处理器（可用 `BlockCapabilityCache`），插入失败后停止尝试该方向 |
| C-P1-6 | `ManaSparkEntity.java:171-180, 465` | 支配火花每 tick 做实体搜索；池子满与不满来回切换时，范围内所有火花都重新搜索，最坏 O(N²) | 每 tick 按（维度、中心、颜色）缓存搜索结果；火花集合变化时才重算 |
| C-P1-7 | `LuminizerBlockEntity.java:116-173` | 每个中继每 tick 都沿整条链走到终点，用 ArrayList 判环，整体 O(N³)；而且不检查区块是否已加载，**可能在主线程同步加载区块**（中等置信度） | 先搜索末影珍珠，找到了才求终点；判环改用 HashSet；每一跳检查 `isLoaded` |
| C-P1-8 | `mixin/ExperienceOrbMixin.java:43-50` | 每个经验球每 20 tick 遍历最多 4 个区块里的所有方块实体，寻找 Rosa Arcana | 每个维度维护已加载的 Rosa Arcana 集合（Bergamute、Solegnolia 已经这样做了） |

### P2

| 编号 | 位置 | 问题 |
| --- | --- | --- |
| C-P2-1 | `ManaBurstEntity.java:866-886` | `fromEntity` 应改用 `getInBlockState()`（原 1.20 语义），目前只在 y 几乎是整数时误判，详见第 0 节 |
| C-P2-2 | `ManaPoolBlockEntity.java:123-130` | 每次魔力变化都 `setChanged()`，带 6–12 次邻居查询；中央池每 tick 可达数十次。改为 `markForPersisting()`，只在比较器输出变化时通知 |
| C-P2-3 | `ThrottledPacket.java:56-82` | 每次判断都遍历所有玩家，而且先算视线方向，再判断手里有没有法杖。应调换顺序，或每 tick 预先算出持法杖的玩家列表 |
| C-P2-4 | `ManaItemHandlerImpl.java`，`CurioIntegration.java` | 每次请求魔力都重扫背包和饰品栏并发事件，每个玩家每 tick 多次。应按玩家每 tick 缓存 |
| C-P2-5 | `RingOfMagnetizationItem.java:73-120` | 先做昂贵的检查（遍历所有维度的 Solegnolia）。应先做便宜的检查 |
| C-P2-6 | `EnderOverseerBlockEntity.java:48-74` | 只要任意维度有一个 Overseer，每个玩家每 tick 做一次 64 格射线 |
| C-P2-7 | `HydroangeasBlockEntity.java:87-91`，`FluidGeneratorBlockEntity.java:82-84` | 先查生物群系再判断是否下雨；空闲时每 tick 分配并打乱 9 个 BlockPos |
| C-P2-8 | `ManaPoolBlockEntity.java:140-200`，`RunicAltarBlockEntity.java:219-240`，`PureDaisyBlockEntity.java:138-150` | 配方查询没有按物品缓存 |
| C-P2-9 | `CorporeaCrystalCubeBlockEntity.java:62-66` | 每个方块每秒统计一次整个网络；客户端收到更新时还会重建区块网格 |
| C-P2-10 | `CorporeaSparkEntity.java:85-112` | 每 tick 重新查找节点并新建节点对象 |
| C-P2-11 | `LivingEntityMixin.java:24-34` | 所有生物每次换方块都去拿完整的饰品容器 |

## 4. 其他 P2（网络或客户端）

- `AgricarnationBlockEntity.java:94-96`：每次催熟成功都发一次粒子事件加一次音效，满魔力时约 17 次/秒/朵。
- `PaintslingerLens.java:96`：每涂一个方块发一个包，一次最多 1000 个。
- `SpectralRailBlock.java:68`：每辆悬浮矿车每 tick 一个粒子事件。
- `RannuncarpusBlockEntity.java:124`：拾取动画的数量符号写反了（客户端会短暂显示变多）。
- 方块实体首次 tick 和绑定时的重复同步：`ManaSpreaderBlockEntity.java:196-199, 663, 737`。
- `ManaPoolBlockEntity.java:401-408`：传输结束时的同步绕过了限流，可能来回抖动。
- `PetalApothecaryBlockEntity.java:104-107`：装岩浆时只要上面有物品，每 tick 都同步（防火物品永远不会烧掉）。
- `CraftyCrateBlockEntity`：每放入一个物品就发整个物品栏。
- `ManaEnchanterBlockEntity`：每 5 tick 发一次完整存档。
- `RedirectiveLens.java:96-118`：每次命中都提交重定向，角度没变也提交。
- 客户端收包后误调用 `setChanged` 或 `sendBlockUpdated`：`SimpleInventoryBlockEntity`、`PlatformBlockEntity`、`BotanicalBreweryBlockEntity`、`HoveringHourglassBlockEntity`。
- `AssemblyHaloItem.java:114-135`：没拿在手上时每 tick 把玩家朝向写进物品。
- `BabylonWeaponEntity`、`ManaPoolMinecartEntity`：每 tick 改实体数据。
- `LuminizerMoverEntity`：追踪范围 10 区块，更新间隔 3。

## 5. 顺带发现的正确性问题（不属于性能）

- `NeoForge/.../xplat/NeoForgeXplatImpl.java:437-458`：`transferEnergyToNeighbors` 查的是自己的位置，而不是邻居。魔力力场（Mana Fluxfield）因此永远不会向外输出能量，还白做 6 次查询。
- `HoveringHourglassBlockEntity.java:157`：同一 tick 内的多次计数会被原版合并成一次。
- `JumpPacket`：服务端不校验跳跃次数，可以用来作弊。

## 6. 已检查、判定无需处理

- 自定义包：
  - 一次性或按动作发送：竞技场指示、天空之杖、黑莲花、歌姬护符、附魔器、芙格尔之眼、GoG、草种、光环合成、物品寿命、烟雾、粒子光束、丰饶之杖、火花网络指示、雷鸣者、剩余物品提示。
  - 已有校验：`LeftClickPacket`。
- 方块实体：
  - 功能花和产能花的基类都走 `ThrottledPacket`，只同步给持法杖的玩家，包也小。
  - 头像、香炉、火花调整器、魔力棱镜、小土豆、酿造台、风箱、光明中继、平台、泰拉凝聚板、精灵门、各种红线方块、虚无伪装类都只在事件时同步。
- 实体：
  - 魔力脉冲的颜色、起始魔力、衰减参数、来源坐标只在生成时设置。
  - 火花和 Corporea 火花是静止的，数据只在交互时变化。
  - 魔法飞弹、荆棘飞轮等写的是相同的值，不会变脏。
  - 盖亚只在阶段切换时改数据。
- CPU：
  - `ManaNetworkHandler` 只在客户端运行。
  - Corporea 节点列表按 tick 缓存。
  - 其他按间隔运行的花（Kekimurus、Rafflowsia、Narslimmus、Clayconia、Jaded Amaranthus、Loonium 等）、脉冲回传搜索、魔力池 1×1×1 物品搜索都没问题。

## 7. 第二阶段进度

### 原则：保持网络协议不变

这一轮修复**只改服务端行为，不改任何网络包的格式**。服务器换新 jar 后，客户端继续用原来的 Botania jar 也能正常连接、正常游戏。客户端没换 jar 时，唯一的区别是视觉效果（粒子数量、头冠计时条按步更新）。

### 已修复（2026-11-22，`dev/phase2-network` 分支）

- **N1 火花粒子包**（`ManaSparkEntity`）
  - 实际没有传输魔力时不发包。
  - 同一对火花每 4 tick 才发一次，不同的火花对错开在不同 tick 发送。
  - 包数量约降为原来的 1/4 或更少。视觉上流动粒子变少。
  - 未做：把一个火花的所有连接合并成一个包、由客户端自己生成粒子（需要改包格式，客户端也要更新），以及支配火花每 tick 重新搜索（会稍微改变抽取时机）。
- **N2 饰品与魔力物品**
  - 新增 `common/impl/mana/DeferredManaCharge`：服务端每 tick 仍按"已欠 + 本 tick"的总额做模拟请求，所以"这一 tick 有没有魔力可用"的判断和原来一致；但实际从魔力物品扣魔力，最多每 10 tick 才做一次。
  - 已接入：芙格尔头冠飞行消耗、隐身斗篷、染血吊坠、脊索之戒。
  - 芙格尔头冠的剩余飞行时间：服务端按玩家保存精确值。只在变化达到 20 tick，或跨过"耗尽 / 满 / 10% 起飞线"时才写回物品。服务端所有判断都读精确值，所以飞行时长不变。
  - 玩家下线时把欠的魔力结清，并把精确的飞行时间写回物品。
  - 与原版的已知差别，都限制在 10 tick 的消耗以内：
    - 结算前拿走付费的魔力物品，欠款从剩下的物品里扣，扣不到就免了。
    - 停用的饰品欠的魔力，要等下次使用或下线时才扣。
    - 同一玩家的两个分批扣费项目，各自只看得到自己的欠款。
  - 未做：受损工具和盔甲的自修复（工具耐久本身每 tick 都在变，分批扣魔力没有收益）、魔力镜（N5）、泰拉粉碎者（N6）、光环之戒、扩散火花给玩家充能、旁观者饰品（N16）。
- **N3 魔力脉冲衰减**（`ManaBurstEntity`）
  - 魔力改存在普通字段里，两端各自模拟衰减（客户端原本就会模拟）。
  - 衰减期间服务端每 10 tick 才校正一次，之前是每 tick 同步。
  - 显式修改魔力（镜片、泰拉之刃等）照常立即同步。

### 第二批（2026-11-22，`dev/phase3-network-cpu` 分支）

同样不改包格式，客户端不用换 jar。

- **N14 Dodge 包**（`DodgePacket`）：先校验戒指和冷却，再播放音效、扣饱食度。冷却期内收到的包直接丢弃。正常客户端只有在服务端同步过来的冷却结束后才会发包，不受影响。
- **N15 Corporea 索引请求**（`IndexKeybindRequestPacket`、`IndexStringRequestPacket`、`CorporeaIndexBlockEntity`）：
  - 同一玩家两次请求之间至少隔 4 tick（0.2 秒），太快的请求直接丢弃。
  - 按键请求的数量限制到 `MAX_REQUEST`，和聊天请求一致。
  - 聊天请求超过 256 个字符时忽略（与原版聊天长度上限一致）。
- **C-P1-7 光明中继**（`LuminizerBlockEntity`）：
  - 先查找有没有末影珍珠，有了才校验绑定、沿链找终点。结果和原来一致。
  - 链上判环改用 identity 集合。
  - 稳态下每 tick 只剩一次很小的实体搜索；只有真的需要传送珍珠时才走链，可能加载区块的情况也只在这时发生。
- **N4 魔力池充能物品**（`ManaPoolBlockEntity`）：物品魔力有变化时，每 10 tick 同步一次；传输在下一 tick 无法继续（物品满了或空了、池子空了或满了）时立即同步。物品没变化（例如创造魔力平板）时不再同步。仅影响视觉。
- **N7 脉冲追踪范围**（`BotaniaEntities`）：从 6 区块降到 4 区块，和原版投射物相同。仅影响视觉：更远处的脉冲不再显示。
- **N8 发射器接收方比较**（`ManaSpreaderBlockEntity`）：改为比较位置和维度。对着魔力虚空，或附属每次都返回新对象的 capability，不再每次查询都触发立即同步。
- 未做：
  - N9 发射器精简同步包：客户端读取时缺失的键会变成默认值，要单独设计读取路径，风险较大。加上 N8 之后它的触发次数已经少很多。
  - N10 转盘发射器按需模拟：改变模拟时机需要测试。

### 第三批（2026-11-22，`dev/phase3-cpu` 分支）：服务端 CPU

只改服务端，不影响客户端。

- **新增 `common/helper/SectionScanHelper`**：
  - 大范围方块扫描改用它。遍历顺序和 `BlockPos.betweenClosed` 完全一致（先 x、再 y、再 z，已对照原版源码核实）。
  - 先用区块段调色板的 `maybeHas` 跳过不可能有匹配方块的段，其余段直接从段里读方块状态。
  - 匹配到的位置和顺序与原来逐格调用 `getBlockState` 完全相同，所以依赖顺序的随机选择结果不变。区块的获取方式也与原来相同。
- **C-P1-1 Bubbell**：每 tick 的整球扫描改用上述 helper，没有水的区块段直接跳过。
- **C-P1-3 Orechid / Marimorphosis / Orechid Ignem**：候选位置扫描改用 helper。没有可转换方块的段直接跳过；候选列表顺序不变，所以随机选中的位置不变。
- **C-P1-4 Munchdew**：树叶扫描改用 helper。随机数抽取的次数和顺序不变。
- **C-P1-2 Exoflame**：
  - 原来对 605 个位置逐个调用 `getBlockEntity`，现在改为读取范围内 1–4 个区块的方块实体表，按原遍历顺序（z、y、x）排序后再处理。
  - 先复制成列表再处理，处理过程中熔炉改变方块状态也不会影响遍历。
  - 已移除的方块实体会被跳过。
- **C-P1-8 经验球与 Rosa Arcana**：
  - Rosa Arcana 在 `clearRemoved` 时加入"已加载"集合，`setRemoved` 时移出。客户端和服务端各用一个集合。
  - 经验球在这个集合里找最近的花，不再每 20 tick 遍历附近最多 4 个区块的所有方块实体。
  - 唯一差别：两朵花与经验球距离完全相等时，选中哪一朵可能不同。
- 未做：
  - C-P1-5 Hopperhock：要改插入重试的逻辑，需要测试。
  - C-P1-6 支配火花：会改变抽取的时机。

### 验证情况

- `Xplat`、`NeoForge`、`Fabric` 三个模块编译通过，`spotlessCheck` 通过（三批都是）。
- 第三批还需要在游戏里验证：
  - Bubbell 的排水范围正常；
  - Orechid / Marimorphosis 的转换正常；
  - Munchdew 吃树叶正常；
  - Exoflame 加速熔炉正常；
  - 经验球会飞向 Rosa Arcana（包括区块重新加载之后）。
- 第二批还需要在游戏里验证：
  - 戒指闪避正常，冷却显示正常；
  - Corporea 索引的聊天请求和按键请求正常；
  - 光明中继传送末影珍珠正常；
  - 魔力池里充能的物品颜色会更新；
  - 发射器对着魔力虚空时绑定显示正常。
- 已打出 `NeoForge/build/libs/botania-neoforge-1.21.1-456-SNAPSHOT.jar`。
- **尚未在游戏里测试**，需要在测试服验证：
  - 火花网络的魔力传输量不变；
  - 斗篷、头冠在魔力即将耗尽时，停止的时机与原来一致；
  - 下线再上线后飞行时间正确；
  - 远距离发射器的脉冲正常到达，粒子大小正常。

## 8. 第二阶段修复建议顺序

1. **N1 火花粒子包**、**N2 饰品与魔力物品改写**、**N3 脉冲衰减同步**：预计收益最大，而且都只涉及视觉或同步方式。
2. N4 魔力池充能物品、N5 魔力镜、N6 泰拉粉碎者、N8 + N9 + N10 发射器同步、N12 符文祭坛。
3. N14、N15 服务端收包校验（防止被恶意刷包）。
4. CPU 的 P1：C-P1-1 到 C-P1-8，其中 C-P1-7 有同步加载区块的风险，优先处理。
5. P2 视工作量顺带处理。

### 落地前需要确认

- 服务器实测：建议用 spark 跑一次 profiler，再统计网络包类型，用来确认第 1 节三项的实际占比。本审计没有实测数据，规模估算都是按假设的基地规模算出来的。
- 服务器上的 Curios 版本，以及魔力戒指是否戴在饰品槽里（决定 N2 的严重程度）。
- 附属检查：MythicBotany、ExtraBotany 等附属给 `ManaReceiver` 注册的 capability 如果每次都返回新对象，会同样触发 N8。
- 构建：这个分支要打出 jar 需要运行 Gradle。为了不影响依赖它的附属，版本号建议仍落在 `[456-SNAPSHOT,457)` 范围内（例如 `456-scex.1`）。但 Timeless-Ivy 和 Just-Enough-Botania 的依赖写的是精确的 `[456-SNAPSHOT]`，需要同时放宽。
