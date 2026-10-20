# NeoForge 模组开发（minecraft-neoforge-mod）

> 前置：同时加载 minecraft-java-build（构建/Gradle/JDK 问题先看它）。本技能覆盖 1.20.1（legacyforge 线）、1.21.x、26.2、26.3 四线。
> **铁律：API 签名一律 `read_file` 查 `references/api/`，禁止凭记忆写签名。** 参考里没有的，用 web 工具查官方文档（docs.neoforged.net / projects.neoforged.net），不要编。
> 核对日期：2026-09（26.3 上线复核：moddev 2.0.147、26.3 仅 `-beta`（26.3.0.12-beta）、Java 25；26.2 线坐标不变）。

## 1. 定位

- NeoForge 是 Forge 的继任者（1.20.1 起分叉，自研加载器与 API；20.2.x 起用独立版本号）。本技能支持四线：
  - **1.20.1**：官方路线是 **legacyforge 插件 + Forge 47.1.3 坐标**（`net.neoforged.moddev.legacyforge` 2.0.91，NeoForge 自研 1.20.1 版本已 EOL，官方 MDK 现状即此形态）；元数据走老式 `mods.toml`（javafml）。
  - **1.21.1（独立钉选线）**：`net.neoforged.moddev` 2.0.144 + NeoForge **21.1.249**；元数据 `neoforge.mods.toml`，但 **必须显式声明 `modLoader="javafml"` + `loaderVersion`**——javafml 默认化从 21.11 起，1.21.1 缺这两个字段客户端直接拒载（`Missing ModLoader`）。模板目录 `neoforge-1.21.1`（与 1.21 线同构，仅这两字段与坐标不同）。
  - **1.21.x（默认主流线，1.21.11）**：`net.neoforged.moddev` 2.0.144 + NeoForge 21.11.45；元数据 `neoforge.mods.toml`（新形态）。
  - **26.2（26.x 前一线，模板钉选）**：`net.neoforged.moddev` 2.0.144 + NeoForge **26.2.0.59**（模板钉选值；maven 上 26.2 已发到 26.2.0.88）。2026-06 起的 26.x 首线，生态仍在跟进（部分模组库滞后）。
  - **26.3（当前 26.x，2026-09-15 发布，仅 beta）**：`net.neoforged.moddev` **2.0.147** + NeoForge **26.3.0.12-beta**。**maven.neoforged.net 上 26.3 目前只有 `-beta` 构建**（26.3.0.0-beta … 26.3.0.12-beta，无任何非 beta 版本，2026-09 实测）——拿 26.3 做生产项目前先确认这个状态；生态未跟上，优先 1.21.11 或 26.2 线。
- moddev（ModDevGradle）= 官方 Gradle 插件，内置 **NeoForm** 反编译管线（把混淆 Minecraft 反编译为官方映射的可用源码），`neoFormVersion` 可切 vanilla 模式。

## 2. 版本对应

| 线 | MC | Gradle 插件（build.gradle plugins） | NeoForge/Forge 坐标（neo_version） | Gradle wrapper | JDK |
|---|---|---|---|---|---|
| 1.20 | 1.20.1 | `net.neoforged.moddev.legacyforge` 2.0.91 | 1.20.1-47.1.3（Forge 47.1.3） | 8.14.5 | 17 |
| 1.21.1 | 1.21.1 | `net.neoforged.moddev` 2.0.144 | **21.1.249** | 9.2.1 | 21 |
| 1.21 | 1.21.11 | `net.neoforged.moddev` 2.0.144 | 21.11.45 | 9.2.1 | 21 |
| 26 | 26.2 | `net.neoforged.moddev` 2.0.144 | 26.2.0.59 | 9.2.1 | 25 |
| 26.3 | 26.3 | `net.neoforged.moddev` 2.0.147 | **26.3.0.12-beta（只有 `-beta`，无正式版）** | 9.2.1 | 25 |

- 新项目一律 `mc_scaffold`（platform=neoforge，可选版本）生成，坐标已按上表钉好；不要手动拼版本。
- 版本核对源：https://projects.neoforged.net （NeoForge 各版本列表）、官方 MDK 仓库（github.com/NeoForgeMDKs）；程序化核对用 maven API：`https://maven.neoforged.net/api/maven/versions/releases/net/neoforged/neoforge`（26.3 只有 `-beta` 即由它实测）与 `https://maven.neoforged.net/releases/net/neoforged/moddev-gradle/maven-metadata.xml`（moddev 插件版本）。moddev README 声明**仅支持 NeoForge ≥21.0.x**——1.20.1 必须用 legacyforge 插件。
- 26.x 线模板（26.2 / 26.3）均无 Parchment 块（官方 MDK 现状）；1.21 线有（Parchment = 社区映射，补方法名/参数名，需接受授权，构建时自动获取）。

## 3. 项目骨架

`mc_scaffold platform=neoforge` 生成的布局（1.21 线）：

```
build.gradle / settings.gradle / gradle.properties / gradle/wrapper/...
src/main/java/<pkg>/<MainClass>.java        # @Mod 主类
src/main/resources/META-INF/neoforge.mods.toml
src/generated/resources/                     # runData 输出（源集已挂载）
```

- `build.gradle` 核心：`neoForge { version = project.neo_version; runs { client / server / data }; mods { "<mod_id>" { sourceSet sourceSets.main } } }`；`neo_version` 来自 `gradle.properties`，与 `minecraft_version` 必须是同一 MC 版本。
- `processResources { filesMatching('META-INF/neoforge.mods.toml') { expand project.properties } }` 把 toml 里的 `${mod_id}` 等占位符展开成实际值（Gradle 侧 `${}` 展开；脚手架侧 `{{}}` 渲染在前，两者不冲突）。**只对 toml 展开**是有意为之：`expand` 底层是 Groovy 模板引擎，会把它看到的任何「美元号+花括号」片段当表达式求值——其它资源文件（乃至 toml 注释）里出现这类字样会让构建以模板解析失败收场。
- 1.20.1 差异：插件 `net.neoforged.moddev.legacyforge`，`legacyForge { version = "1.20.1-47.1.3" }`，元数据文件是 `META-INF/mods.toml`（老形态），有 `pack.mcmeta`。
- 26.x 差异（26.2 / 26.3 相同）：仅 Java toolchain 25、无 Parchment 块，其余同 1.21。

## 4. neoforge.mods.toml 必填字段

**21.11+ / 26.x** 形态（无 `modLoader`/`loaderVersion`，走默认 javafml；**1.21.1 是例外**——必须显式声明，见下方第 3 条）：

```toml
license="All Rights Reserved"   # 必填（可改 MIT 等开源许可证）

[[mods]]
modId="my-plugin"               # 必填；^[a-z][a-z0-9_]{1,63}$
version="${mod_version}"        # 建议必有（gradle.properties 展开）
displayName="${mod_name}"       # 建议必有
authors="${mod_authors}"        # 建议必有
description="${mod_description}"

[[dependencies.my-plugin]]      # [[dependencies.<modId>]]
modId="neoforge"                # 必填依赖
versionRange="${neo_version_range}"
mandatory=true

[[dependencies.my-plugin]]
modId="minecraft"
versionRange="${minecraft_version_range}"
mandatory=true
```

- 1.20.1 时代（`mods.toml`）：另有必填 `modLoader="javafml"`、`loaderVersion`（如 `"[47,)"` 对应 Forge 47 加载器）；依赖段 `modId="forge"` + `minecraft`。
- **1.21.1 例外（neoforge.mods.toml 也要这两个字段）**：`modLoader="javafml"` + `loaderVersion="[1,)"`。1.21.1 的 FML 4.0.x 尚未把 javafml 设为默认，缺失则客户端拒载并报 `Missing ModLoader`；21.11 起才可不写。`mc_scaffold` 的 1.21.1 线模板已内置这两个字段。
- 陷阱：modId 非法字符/大写（`^[a-z][a-z0-9_]{1,63}$`）、dependencies 表头写成 `[[dependencies]]`（必须带 `.<modId>`）都会导致加载失败。
- 与 `@Mod` 注解的 modid 必须一致——**gradle.properties 的 `mod_id` 和 `@Mod(MyPlugin.MODID)` 两处都要改**（脚手架生成时已一致，改模组名时注意两处）。

## 5. API 要点（签名一律查 references/api/）

- **入口与事件总线**：`@Mod` 类 + 构造器注入 mod bus（`IEventBus`）；game bus = `NeoForge.EVENT_BUS`；`@SubscribeEvent` / `@EventBusSubscriber(bus = Bus.MOD|GAME)` → `references/api/mod-bus-events.md`
- **注册**：`DeferredRegister.create(Registries.ITEM, MODID)` + `register(modEventBus)` + `DeferredHolder` → `references/api/deferred-register.md`（1.20.1 线返回 `RegistryObject`）
- **数据生成**：`GatherDataEvent`（mod bus）+ `DataGenerator.addProvider` + runData → `src/generated/resources` → `references/api/data-generation.md`
- **MC 26.2 → 26.3 移植变更（Architectury API 21.1 → 22）**：`LootEvent`、扩展菜单（`ExtendedMenuDataProvider`）、`BlockTransformerHooks`、`FuelRegistry`、`BiomeHooks`、`GenerationProperties` → `references/api/changes-26.3.md`（**用 Architectury 的工程写 26.3 前必读**。来源是 Architectury 官方移植指南，**不是** NeoForge 核心 API 的变更清单；未核实的条目标 `UNVERIFIED`）
- 客户端/服务端隔离：`@EventBusSubscriber(dist = Dist.CLIENT)` / `@OnlyIn(Dist.CLIENT)`（见 mod-bus-events.md 第 4 节）。

## 6. 构建 / 测试流程

1. `gradlew build`（Windows: `cmd /c "gradlew.bat build"`）→ 产物 `build/libs/<name>-0.1.0.jar`。
2. 首次构建下载 Gradle + JDK toolchain + 依赖，**NeoForm 反编译可达一小时**（R9）——设足够超时，别以为卡死。
3. `gradlew runClient` 起开发客户端（带模组）；`gradlew runServer` 起开发服务器（首次会生成 `run/eula.txt`，改为 `eula=true`；服务器默认 `online-mode=true`，局域网/单人调试可改 `online-mode=false`）。
4. `gradlew runData` 生成数据文件（见 data-generation.md）。
5. 发布：`build/libs/<name>-0.1.0.jar` 拖入客户端 `mods/` 目录（客户端模组直接进 mods/；服务端模组进服务器 mods/）。

## 7. 常见坑

1. **moddev 与 NeoForge 版本配对**：moddev 2.x 仅支持 NeoForge ≥21.0.x；1.20.1 用 `legacyforge` 插件（`net.neoforged.moddev.legacyforge`），插件选错 → 构建或加载直接失败。
2. **26.3 没有正式版**：maven.neoforged.net 上 26.3 全是 `-beta`（最新 26.3.0.12-beta），生态未跟上；26.2 已有非 beta 构建。要在 26.x 做生产开发，用 26.2 或 1.21.11 线；用 26.3 就要接受 beta 风险，且 26.2 → 26.3 有移植变更（详见 `references/api/changes-26.3.md`，来源为 Architectury 移植指南）。
3. **`minecraft_version` 与 `neo_version` 必须对应同一 MC**（如 `1.21.11` ↔ `21.11.45`），改版本只动一处 → 构建产物加载即崩。
4. **modid 三处一致性**：`gradle.properties` 的 `mod_id`、`@Mod(...)` 注解、`neoforge.mods.toml` 的 `modId`（后两者由脚手架渲染一致；手改注意同步）。
5. **`${}` 与 `{{}}` 双展开**：toml 里 Gradle 占位符是 `${mod_id}`（build 时展开），脚手架占位符是 `{{name}}`（生成时展开）；不要把 `${...}` 写成 `{{...}}`（脚手架会报未定义占位符），反之亦然。
6. **生成文件被覆盖**：`src/generated/resources` 是 runData 的输出，改 provider 而不是改输出文件。
7. **runServer 首次失败**：先查 `run/eula.txt` 是否 `eula=true`；`online-mode` 调试期可关。
8. **首次构建超时**：NeoForm 反编译 + Parchment 下载可达一小时，日志无输出不代表卡死。
9. **1.20.1 线是老形态**：`mods.toml`（非 neoforge.mods.toml）、`modLoader="javafml"`、依赖 `forge` 段——照 3.8/4 节 1.20.1 差异处理，别套 21.x 形态。
10. **1.21.1 缺 modLoader → `Missing ModLoader`**：把 1.21.11 的 toml 直接用到 1.21.1 上必然拒载（javafml 默认化从 21.11 起）。1.21.1 必须写 `modLoader="javafml"` + `loaderVersion="[1,)"`。
11. **Groovy expand 陷阱**：`processResources` 的 `expand` 会把资源文件里任何「美元号+花括号」片段当表达式求值——即使写在 toml 注释里也会让首次构建失败。只对需要的文件展开，别在资源文件里写这类示例文字。
12. **请求未钉选的完整版本号会响亮失败**：`mc_scaffold` 对 fabric/forge/neoforge 三平台要求版本精确命中钉选值（防"想要 1.21.1、拿到 1.21.11"的静默替换）；报错会列出可用版本。Paper/Spigot 仍宽松（具体版本只影响 api-version 推导）。

## 8. 开工前核对（intake）

- 必问：**线**（1.20.1 legacyforge / **1.21.1** / 1.21.x（1.21.11）/ 26.2 / 26.3（**目前只有 `-beta`**）——mods.toml 与 neoforge.mods.toml 不同，1.21.1 还需显式 modLoader）、neo 版本、是否客户端/服务端/双端。
- 用户信息不足先批量提问（minecraft-intake），禁止猜着开工；"你决定"→ 默认 1.21.x 主线（26.x 中 26.3 仅 `-beta`、26.2 生态仍在跟进，二者都别默认）。

## 9. dev 测试（把游戏跑起来验证）

构建通过 **不等于** 能跑：`@Shadow` 字段类型写错、缺少运行时前置依赖、mixin 目标方法不存在，
javac 一个都不报，只在启动那一刻炸。所以交付前要真起 `runServer` + `runClient` 验一遍。

**驱动 dev 客户端必须用线程输入挂接** —— `keybd_event` / `SendInput` 默认无效，GLFW 只认真正持有焦点的
窗口；要 `AttachThreadInput(自身线程, MC线程, true)` + `SetForegroundWindow` + `SetFocus`，断开挂接后再发键。
自检判据：挂接后按 F3 能出调试屏。

`mcdev_*` 探针工具是**平台绑定**的（为某个 Forge/NeoForge 版本 dev-mapped 构建），换大版本就装不上——
1.20.1 的 probe 装不进 1.21.1。这时不要反复重试 `mcdev_prepare`，走手工流程。

NeoForge 这里的三条专属细节：run 配置用 `gameDirectory` 分开客户端/服务端；`runGameTestServer` 在没注册
gametest 时会抛 `No test functions were given!`（假阴性，用 `runServer`）；`runServer` 需要
`net.neoforged:DevLaunch`，`maven.neoforged.net` 不通时要给 Gradle 显式加代理参数。

完整流程（输入挂接代码、截图判读、首启无障碍页坑、5 条验收信号、收尾）见
`../minecraft-java-build/references/dev-test.md`（该文档覆盖 Paper/Spigot/Fabric/Forge/NeoForge 全平台）。

## 10. Forge 1.20.1 → NeoForge 1.21.1 跨分支同步

同一模组的两个版本常被维护成**互不相干的孤儿分支**：1.21.1 往往是从「初始化移植分支」重开的新根，
与 1.20.1 **没有共同祖先**。这种仓库里 `git merge-base` 返回空、`git diff main origin/1.21.1`
会把 400+ 个文件全算成改动——**完全不能用来找差异**。

正确做法：按「**文件存在性 + 提交语义**」比对。

1. 列两边 `src` 的文件清单做差集。真缺口通常只有寥寥几个文件（一边有一边没有），这几个才是重点。
2. 逐条读上游提交的 diffstat，判断该提交的**语义**是否已在目标侧以**另一种实现**落地过
   —— 移植时提交信息常被重写，不能靠 `git log` 里的文字对上号。
3. 最后才看具体文件内容差异。

**API 形态翻译表**（内容照搬，API 形态绝对不能照搬）：

| 1.20.1 Forge | 1.21.1 NeoForge |
|---|---|
| `ForgeConfigSpec` / `ForgeConfigSpec.Builder` | `ModConfigSpec` / `ModConfigSpec.Builder` |
| `addAttributeModifier(attr, uuidStr, v, MULTIPLY_BASE)` | `addAttributeModifier(attr, ResourceLocation, ADD_MULTIPLIED_BASE, amplifier -> …)` |
| 覆写 `getAttributeModifierValue(amplifier, modifier)` | 上面那条 `amplifier -> …` 曲线（Forge 那个覆写的默认实现本就是 `amount * (amplifier+1)`，所以等价） |
| `ForgeRegistries.MOB_EFFECTS.getKey(x)` | `BuiltInRegistries.MOB_EFFECT.getKey(x)` |
| `Map<MobEffect, MobEffectInstance> activeEffects` | `Map<Holder<MobEffect>, MobEffectInstance>` |
| `ModConfigEvent`（`net.minecraftforge.fml.event.config`） | `net.neoforged.fml.event.config.ModConfigEvent`（有 `Loading`/`Reloading`/`Unloading`） |
| `CommandSourceStack` / `mods.toml` 时代写法 | `neoforge.mods.toml`（1.21.1 还需 `modLoader="javafml"`） |

最后一条同理的坑：`@Shadow` 字段**类型**写错（`Map<Holder<MobEffect>, …>` 写成 `Map<MobEffect, …>`）
编译期完全不报错，只在运行时 mixin 应用阶段炸。跨分支同步后必须真跑一次 dev 测试（见 §9）。

### 10.1 逐项核对清单（漏一项就是一次返工）

按顺序过，每项都写清「怎么验」：

- [ ] **版本号**：`mod_version` 常被藏在**功能提交**里，不是只有 `build:` 前缀的提交才会改。
  实测遇到过「新增某效果」的提交顺手把 1.4.19 升到 1.4.20，只核 `build:` 提交就会漏一版。
  直接比两侧文件，别靠提交信息猜：
  `Select-String -Path <repo>\gradle.properties -Pattern '^mod_version'`。
  目标侧产物名是 `mod_version+minecraft_version` 拼接（如 `tcc-1.4.20+1.21.1.jar`），
  改完记得确认包内 `neoforge.mods.toml` 的 `version=` 也跟着变了，并删掉 `build/libs` 里的旧产物。
- [ ] **依赖坐标与级别**：上游 `build.gradle` 每条依赖的 maven 坐标 / fileId、以及
  `implementation` ↔ `compileOnly` 的**升降级**都要一条条对。等级一变，会连带要求改
  `*.mods.toml` 的 `[[dependencies]]` 段，以及在主类加 `ModList.get().isLoaded(...)` 运行期守卫。
- [ ] **mods.toml 依赖段**：新增/删除的 `modId`、`mandatory`、`versionRange` 逐条对齐。
- [ ] **新类的注册点**：新增的客户端类要在主类的 `Class.forName(...)` 块里登记，
  否则只在客户端崩、服务端看不出来。
- [ ] **lang 键**：`zh_cn.json` 和 `en_us.json` 两侧都要加。但**先确认语义是否已在目标侧用别的键实现**
  —— 例：上游给 `attributeslib.modifier.plus/take` 加兜底，而 1.21.1 侧 `BaseCurioItem`
  早已改用 `neoforge.modifier.plus/take`，盲目搬过来就是重复且误导。
- [ ] **mixins 配置**：新增 mixin 类要加进 `*.mixins.json` 的 `mixins` 数组，`compatibilityLevel` 与 JDK 对齐。
- [ ] **数据文件里的硬编码 modid**：affix / advancement / loot / recipe 等 JSON 里写死的引用
  （如 `attributeslib:crit_chance`），换依赖后必须跟着换。这类错编译期发现不了，只在运行期静默失效。
- [ ] **默认值与配置键**：`defineInRange` 的默认值变化、新增的配置项，会让服务端日志出现
  `<modid>-common.toml is not correct. Correcting` —— 这是**预期**的配置迁移，不是故障。
- [ ] **删除的文件**：上游删掉的类在目标侧也要删，然后全局搜残留引用（渲染器 / Handler / 注册表 / 资源文件）。

### 10.2 三条实证坑

1. **`MobEffectInstance.duration` 在 1.21 已是私有字段且无 setter**。1.20.1 里
   `instance.duration = n` 的写法到 1.21 编译不过；`mapDuration()` 只读不写。要压时长可用：
   构造同参数新实例 + `forceAddEffect(newInst, null)`（别用 `removeEffect`，若该效果有保护守卫会被拦）。
2. **网络体系整体换了**：1.20.1 的 `SimpleChannel` / `registerMessage`，在 1.21.1 侧往往已迁到
   NeoForge Payload（`CustomPacketPayload` + `StreamCodec` + `registrar.playToServer/playToClient`，
   发送用 `PacketDistributor.sendToServer/sendToPlayer`）。新包要照**目标侧既有包**的形态写，
   不能照搬 1.20.1 的 `record` + `Supplier<NetworkEvent.Context>` + `enqueueWork`。
   同理：`TickEvent.ClientTickEvent`（带 phase）→ `ClientTickEvent.Post`；
   `LivingHurtEvent` → `LivingIncomingDamageEvent`；`Tesselator.getBuilder()+DEBUG_LINES` 直连绘制
   → 自研 `RenderType`（`DEBUG_LINES` + `NO_DEPTH_TEST` + `COLOR_WRITE`）走 `MultiBufferSource`。
3. **第三方 jar 的元数据可能坏在编码上**——这不是构建配置问题，别在本地反复折腾。
   现象：FML 报 `File ... is not a valid mod file`，堆栈是
   `com.electronwill.nightconfig.core.io.ParsingException: Failed to parse data from Reader`。
   根因：该 jar 的 `META-INF/neoforge.mods.toml` 含**非法 UTF-8 字节**（中文注释被写成 GBK，
   留下裸 `0xE9` 之类），NightConfig 拒绝解析 → FML 判定整个 mod 无效 → pre-loading 阶段直接失败。
   定位方法：用 `System.IO.Compression.ZipFile` 取出该 toml 的字节，
   再用 `new UTF8Encoding($false, $true)` 严格解码，抛异常即坐实；ASCII 段正常、
   只有注释行报错时，基本就是作者打包时用了非 UTF-8。
   处置：换一个 fileId（先用同法验证新 jar），或让作者修。
   临时绕过只能是「本地重打包 + 只改注释编码」，且必须在构建脚本里写清原因与回退方式。

### 10.3 同步完成的判据

`gradlew build` 通过**只证明能编译**。必须再跑 dev 服务端 + 客户端（见 §9），并在日志里确认：

- `Found valid mod file <jar> with {...} mods - versions {...}`：每个依赖 jar 都被判为有效
  （这一行缺失或变成 `not a valid mod file`，就是 §10.2 第 3 条）
- `Preparing <modid>.mixins.json (N)`：N 与 `*.mixins.json` 里的类数一致
- 每个新 mixin 都有 `Mixing XxxMixin from ... into <目标类>`，
  且注入方法没有 `has 0 overrides` / `won't be passed a CallbackInfo` 之外的异常提示
- 客户端日志出现 `Minecraft NeoForge* <版本> - Multiplayer`，服务端出现 `<玩家> joined the game`
- 全程日志里**自己的包名一次都不出现在异常堆栈中**（这条比「没有 ERROR 字样」可靠得多，
  因为别的 mod 的良性警告会混进 ERROR 关键字）
