# Rage Mod For Java 🚀

> 🚧 **Work In Progress (WIP) - 采用 RDD (README-Driven Development) 模式** 🚧
> 本项目目前处于**早期架构设计与原型开发阶段**，API 随时可能发生破坏性变更。本 README 作为项目的核心设计蓝图，后续代码将严格依照此文档的
> Roadmap 进行实现。暂不提供稳定 Release 版本。

一个面向 RAGE 引擎（GTA V / GTA VI / RDR2）的跨代际模组开发框架，旨在打破传统 C/C++ 或 C# 的开发壁垒，允许开发者使用 **Java
** 语言编写游戏模组（Mod）。本项目通过桥接游戏底层 Hook SDK 与 Java 虚拟机，并**全面采用 Java 25 的 Foreign Function &
Memory (FFM) API**，实现了 Java 代码对游戏 Native 函数的安全、高效调用与内存操作。

---

## 🗺️ 开发路线图 (Roadmap)

*作为 RDD 的核心，以下是本项目的开发任务清单。欢迎在 Issue 中讨论或提交 PR 协助完成：*

### Phase 1: 基础设施与 C++ 桥接层 (当前阶段)

- [x] 确定整体 Monorepo 架构设计与模块划分
- [x] 梳理 ScriptHookV 加载链路与内存交互机制
- [ ] 搭建 `shv-hotspot-loader` 基础 CMake 工程（配置 `CMAKE_CXX_STANDARD 20`）
- [ ] 实现 C++ 侧注入并启动 HotSpot JVM 的最小化原型 (成功打印 Hello World)
- [ ] 搭建 `shv-graalvm-loader` 基础 CMake 工程（配置 `CMAKE_CXX_STANDARD 20`）并实现 DLL 加载逻辑

### Phase 2: Java 核心 API 封装 (`api-framework` 基于 FFM)

- [ ] 编写纯 Java 源码（无 Gradle/Maven，仅 `.java` 文件）
- [ ] 使用 `jextract` 工具从 ScriptHookV 的 C/C++ 头文件自动生成 FFM 绑定代码
- [ ] 封装基础数学与游戏类型 (基于 FFM 的 `MemorySegment` 和 `MemoryLayout`)
- [ ] 封装核心 Native 函数 (映射 `natives.h`，使用 `Linker` 和 `MethodHandle`)
- [ ] 实现游戏主循环 (Game Tick) 到 Java 线程的事件分发机制

### Phase 3: GraalVM 原生编译支持

- [ ] 配置 **GraalVM 25** Native Image 编译流程
- [ ] 解决 FFM API 在 AOT (Ahead-of-Time) 编译下的兼容性与反射限制
- [ ] 跑通首个 GraalVM 编译的 `xxx-script.dll` 并在游戏中加载

### Phase 4: 开发者体验与发布

- [ ] 编写开发者文档与 API 示例
- [ ] 提供一键生成 Mod 模板的 CLI 工具
- [ ] 发布首个 Alpha 测试版

---

## 📐 架构设计 (Architecture Design)

### 📁 模块命名约定

- **Root (`rage-mod-java`)**: 代表项目的整体品牌与跨游戏版本愿景。
- **Core (`api-framework`)**: 纯 Java 源码集合，无构建工具依赖，作为所有 Java Mod 的核心 API 框架。
- **Loaders (`{sdk}-{runtime}-loader`)**: 特定于某个游戏 Hook SDK 与 Java 运行时的 C++ 桥接实现。`shv-` 前缀仅代表当前默认支持的
  GTA5 ScriptHookV 后端，未来将按需扩展其他 SDK 适配器。

### 🌟 包名命名规范 (Package Naming)

框架层 (api-framework)：统一使用 dev.ragemod.api.*作为包名前缀，确立官方核心地位。

Mod 业务层 (xxx.jar)：绝不强制统一包名。要求 Mod 开发者使用反向域名或个人标识作为前缀（如 dev.jaysen.superjump.*或
io.github.username.mymod.*），从根源上杜绝多 Mod 共存时的类名冲突。

### 🎁 JAR 内部目录结构

无论 Mod 开发者的业务代码放在哪个包下，其导出的 xxx.jar 必须遵循以下资源与入口约定：

* 📱 注解驱动的入口扫描
    * Mod 开发者无需将主类放在固定的包路径下。只需在任意包下的主类上添加 @RageModEntry 注解，
    * 框架即可通过反射自动扫描并加载 Mod 入口点，实现“零配置”热插拔。

```text
📁 dev/ragemod/script/               —— Mod 脚本包路径
   📄 xxx.class                      —— 编译后的 Mod 主类
📁 resource/                         —— 资源文件目录
    📁 texture/                      —— 纹理资源目录
```

### 📦 核心框架模块 (Framework)

本仓库采用 Monorepo 结构，包含以下三个核心子模块：

#### 1. `shv-hotspot-loader` (C/C++ 桥接层)

* **核心职责**：作为传统 Java 运行时的桥梁，负责操作 GTA5 的 `ScriptHookV.dll`。
* **主要产出**：`RageMod-HotSpot-Loader.asi`
* **技术特性**：
    * 内嵌并启动 HotSpot 虚拟机（JVM）。
    * 为 `api-framework` 提供游戏底层交互能力，暴露自定义封装的 Native 函数基础设施。
    * 使用 **C++20** 标准编写，提升代码安全性与可维护性。

#### 2. `shv-graalvm-loader` (C/C++ 桥接层)

* **核心职责**：作为 GraalVM Native Image 编译产物的桥梁，操作 GTA5 的 `ScriptHookV.dll`。
* **主要产出**：`RageMod-GraalVM-Loader.asi`
* **技术特性**：
    * 提供游戏底层交互及 Native 函数封装基础设施。
    * 负责直接加载由 GraalVM 编译出的自定义 `xxx-script.dll`，实现免 JVM 环境的原生级运行。
    * 使用 **C++20** 标准编写，优化编译效率与内存管理。

#### 3. `api-framework` (Java 核心 API)

* **核心职责**：整个框架的 Java 侧核心源码集合，所有 Java Mod 均需引用此项目源码。
* **主要产出**：纯 `.java` 源文件（包名 `dev.ragemod.api.*`）
* **技术特性**：
    * **零构建工具依赖**：不包含 Gradle/Maven，仅作为纯 Java 源码存在，由 Loader 或 Mod 项目自行编译引用。
    * **全面拥抱 FFM API (Project Panama)**：基于 **Java 25** 彻底抛弃传统的 JNI/JNA，使用纯 Java 实现跨语言调用。
    * 完整部分封装 RAGE 引擎的数据类型（利用 `MemoryLayout` 精确映射 C++ 结构体），设计上兼容 GTA5/GTA6。
    * 包装并映射游戏底层的 Natives 函数，提供面向对象的语义化 Java API（如 `GameWorld.spawnVehicle()`），使 Mod 代码与底层
      SDK 解耦。

---

## 🛠️ 模组 (Mod) 开发与运行模式

开发者在编写具体的 Java 模组（`xxx-script`）时，可根据需求选择以下两种运行模式：

### 模式 A：基于 HotSpot 虚拟机 (传统 JAR 模式)

* **依赖**：`api-framework` 源码（需 **Java 25** 环境）
* **产出**：`xxx.jar`
* **适用场景**：适合开发期快速调试，或依赖复杂 Java 生态（如反射、动态代理）的模组。

### 模式 B：基于 GraalVM (Native DLL 模式)

* **依赖**：`api-framework` 源码（需 **GraalVM 25** 环境）
* **产出**：`xxx.dll`（通过 `native-image` 命令编译产出）
* **适用场景**：适合追求极致启动速度、低内存占用及防反编译保护的最终发布版本。

---

## 🔄 运行时加载链路 (Lifecycle)

### 🔗 HotSpot 模式加载流程

1. 游戏启动，加载 `dinput8.dll`（ASI Loader 核心）。
2. ASI Loader 注入并加载 `ScriptHookV.dll`。
3. `ScriptHookV.dll` 扫描并加载 `RageMod-HotSpot.asi`。
4. ASI 加载器定位并加载 `RageModHotSpot/jre-minimal/*/jvm.dll`，启动 **Java 25** JVM 环境。
5. JVM 初始化并加载 `RageModHotSpot/mods/api-framework.jar`，执行 初始化 逻辑。
6. 框架初始化并加载 `RageModHotSpot/mods/xxx.jar`，执行 Mod 逻辑。

### 🔗 GraalVM 模式加载流程

1. 游戏启动，加载 `dinput8.dll`（ASI Loader 核心）。
2. ASI Loader 注入并加载 `ScriptHookV.dll`。
3. `ScriptHookV.dll` 扫描并加载 `RageMod-GraalVM-Loader.asi`。
4. ASI 加载器直接加载位于 `RageModGraalVM/` 文件夹下的 `xxx-script.dll`（由 **GraalVM 25** 编译），执行 Mod 逻辑。

---

## 📂 部署目录结构 (游戏根目录)

以下是玩家/开发者部署该框架及模组时，游戏根目录的完整文件树预测：

```text
📄 dinput8.dll                       —— [核心] ASI Loader (负责拦截并加载 .asi 文件)
📄 ScriptHookV.dll                   —— [核心] 第三方 SDK (提供 Native 调用与内存操作)
📄 xinput1_4.dll                     —— [备选] GTA5 增强版环境下的 ASI Loader 替代文件
📄 RageMod-HotSpot-Loader.asi        —— [桥接] HotSpot 虚拟机加载器
📄 RageMod-GraalVM-Loader.asi        —— [桥接] GraalVM DLL 加载器
📁 RageModHotSpot/                   —— HotSpot 运行环境目录
   📁 jre-minimal/                   —— 精简版 JRE 环境 (需包含 Java 25)
   📁 mods/                          —— Java Mod (JAR) 存放处
      📄 api-framework.jar           —— Java API 框架实现
      📄 xxx.jar                     —— 你的 Java 模组
      📄 xxx.java                    —— 你的 Java 源文件
📁 RageModGraalVM/                   —— GraalVM 运行环境目录
   📄 xxx.dll                        —— 由 Native-Image 编译出的原生 Mod
📄 (其他 GTA V 原始游戏文件...)       —— 游戏本体文件
```

---

## 💡 附加技术说明

1. **`ScriptHookV.dll` 的本质**：
   它是 GTA V 模组生态的基石（第三方 SDK），通过进程注入技术 Hook 游戏内核，提供 `nativeInit` 和 `nativeCall`
   等接口，允许外部代码安全地读写游戏内存和调用原生函数。

2. **`dinput8.dll` 的作用**：
   通常作为 ASI Loader 存在。它利用 DirectX 输入库的加载机制，在游戏启动早期劫持进程，从而负责将 `ScriptHookV.dll` 和各种
   `.asi` 插件注入到游戏进程中。

3. **FFM API 与性能优化**：
   本框架采用 **Java 25 的 FFM API** 进行跨语言调用。相比传统 JNI，FFM 提供了更安全的内存访问（通过 `Arena`
   管理生命周期，杜绝内存泄漏）和极低的调用开销。在封装高频调用的 Native 函数时，建议缓存 `MethodHandle` 实例，并结合
   ScriptHookV 的 `scriptRegisterAdditionalThread` 机制，在 Java 层实现合理的多线程与 Tick 循环（Game Loop）管理，避免阻塞游戏主渲染线程。

4. **C++20 特性增强**：
   桥接层采用 **C++20** 标准开发（`CMAKE_CXX_STANDARD 20`），利用 `concepts`、`coroutines`、`modules`
   等现代特性提升代码健壮性，同时兼容现代编译器的优化，确保跨平台与注入场景下的稳定性。

5. **GraalVM 25 AOT 编译优势**：
   使用 **GraalVM 25** 的 Native Image 功能，可将 Java 代码直接编译为原生二进制文件，显著降低启动延迟和内存占用，并增强代码隐蔽性（防反编译），是发布最终
   Mod 产品的首选方案。

---

## 🙏 致谢

感谢以下项目与社区为本框架提供的技术支持与灵感：

* **ScriptHookV**：由 Alexander Blade 开发，是 GTA V 模组开发的基石。
* **Project Panama (OpenJDK)**：为 Java 带来了现代化的 FFM API，彻底改变了 Java 与 Native 代码交互的方式。
* **GraalVM**：Oracle 提供的多语言虚拟机，为高性能原生编译提供支持，优化了 AOT 场景下的反射与 FFM 处理。
* **OpenJDK**：Java 生态的核心开源实现，不断演进的 Java 25 提供了更强大的底层能力。
* **GTA V Modding Community**：无数开发者与测试者为本项目提供了宝贵反馈。

---

## 📜 许可证

本项目采用 [MIT License](LICENSE) 开源协议。你可以自由地使用、修改和分发本框架的代码，包括用于商业性质的 Mod
开发，但请在你的项目声明中保留本项目的版权信息。

**免责声明**：本项目严重依赖第三方 SDK `ScriptHookV.dll`。使用本框架开发 Mod 时，请务必同时遵守 ScriptHookV
官方的最终用户许可协议（EULA）及 Rockstar Games 的 Modding 政策。本项目作者不对因使用本框架导致的任何游戏封号或内存崩溃负责。
