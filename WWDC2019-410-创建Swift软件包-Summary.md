# WWDC 2019 Session 410: 创建 Swift 软件包（Creating Swift Packages）

## 视频信息

- **视频标题**: Creating Swift Packages（创建 Swift 软件包）
- **会议**: WWDC 2019
- **Session编号**: 410
- **发布日期**: 2019年6月6日
- **视频链接**: https://developer.apple.com/cn/videos/play/wwdc2019/410
- **PDF幻灯片**: https://devstreaming-cdn.apple.com/videos/wwdc/2019/410p24ercmpgj258x/410/410_creating_swift_packages.pdf?dl=1

---

## 一、为什么要使用 Swift 软件包 `[0:00 - 2:00]`

不论你是要发布代码来在社区中分享，还是只想能方便地整理 app 中的代码，Swift 软件包都可以为你助力。在 WWDC 2019 之前，Swift Package Manager（SPM）主要用于服务端 Swift 开发和命令行工具；从 Xcode 11 开始，Apple 正式将 SPM 集成到 Xcode 中，使其成为所有 Apple 平台上管理代码依赖和模块化开发的一等公民。

**核心优势包括：**

1. **代码解耦**: 将公共功能模块从主应用中分离出来，降低耦合度，使代码结构更清晰
2. **减少冗余**: 通过创建共享框架，避免在多个目标中重复相同代码（如 iOS 和 macOS app 共享网络层）
3. **跨平台复用**: 一个软件包中的代码可在 iOS、macOS、watchOS、tvOS 之间共享

---

## 二、创建本地软件包 `[2:00 - 6:00]`

### 2.1 在 Xcode 中创建

在 Xcode 11 中创建 Swift 软件包：

1. 选择 **File > New > Swift Package...**
2. 为软件包命名（例如 `MyPackage`）
3. 选择存储位置
4. 勾选 **"Create Git repository on my Mac"** 以初始化版本控制

创建完成后，Xcode 自动生成标准目录结构：

```
MyPackage/
├── Package.swift          # 软件包清单文件（核心配置）
├── Sources/
│   └── MyPackage/         # 主要源代码目录
│       └── MyPackage.swift
└── Tests/
    └── MyPackageTests/    # 测试代码目录
        └── MyPackageTests.swift
```

### 2.2 SPM 支持的最低平台版本

从 Xcode 11 开始，SPM 支持所有 Apple 平台：

| 平台 | 最低版本 |
|------|---------|
| macOS | 10.15 |
| iOS | 13.0 |
| watchOS | 6.0 |
| tvOS | 13.0 |

**注意**: 软件包编译出的代码可以被回溯部署到更早的系统版本上，即 SPM 工具本身需要 macOS 10.15 运行，但编译出的框架在 iOS 12 等旧版本上依然可用（前提是代码兼容）。

---

## 三、软件包清单文件（Package.swift） `[6:00 - 10:00]`

### 3.1 清单文件的作用

`Package.swift` 是整个软件包的**核心配置文件**，定义了软件包的名称、目标、产品、依赖关系以及平台要求。Xcode 和 SPM 通过读取这个文件来了解如何构建、测试和使用这个软件包。

### 3.2 基本结构

最简单的 `Package.swift`：

```swift
// swift-tools-version:5.1
import PackageDescription

let package = Package(
    name: "MyPackage"
)
```

**各部分含义：**

- **`// swift-tools-version:5.1`**：第一行，声明编译此软件包所需的 Swift 工具链最低版本。这不是普通注释，而是 SPM 用来判断如何解析清单文件的指令
- **`import PackageDescription`**：导入 Xcode 提供的特殊库，包含 `Package`、`Target`、`Product` 等 API
- **`Package(name: "MyPackage")`**：主配置入口，`name` 指定软件包名称

### 3.3 完整清单文件示例

一个功能完善的清单文件：

```swift
// swift-tools-version:5.1
import PackageDescription

let package = Package(
    name: "MenuDownloader",
    platforms: [
        .macOS(.v10_15),
        .iOS(.v13)
    ],
    products: [
        .library(
            name: "MenuDownloader",
            targets: ["MenuDownloader"]
        ),
    ],
    dependencies: [
        .package(url: "https://github.com/jpsim/Yams", .upToNextMajor(from: "2.0.0")),
    ],
    targets: [
        .target(
            name: "MenuDownloader",
            dependencies: ["Yams"]
        ),
        .testTarget(
            name: "MenuDownloaderTests",
            dependencies: ["MenuDownloader"]
        ),
    ]
)
```

---

## 四、Target 与 Product `[10:00 - 14:00]`

### 4.1 Target（目标）

Target 是软件包的**基本构建块**。一个 Target 可以定义一个**模块**（module）或一个**测试套件**（test suite）。

- 常规 Target 的源文件位于 `Sources/<TargetName>/` 目录下
- 测试 Target 的源文件位于 `Tests/<TargetName>/` 目录下

**定义 Target：**

```swift
targets: [
    .target(name: "MyTarget"),
]
```

Xcode 会自动查找对应目录下的所有 Swift 文件，**无需手动列出每个文件**。

**自定义路径：**

```swift
.target(name: "FloatingPanel", path: "Framework/Sources"),
```

这在迁移已有项目到 Swift Package 格式时特别有用。

**Target 之间的依赖：**

```swift
targets: [
    .target(name: "MyAppCore", dependencies: []),
    .target(name: "MyAppUI", dependencies: ["MyAppCore"]),
]
```

`MyAppUI` 中的代码可以 `import MyAppCore`。

**测试 Target：**

```swift
targets: [
    .target(name: "MyTarget"),
    .testTarget(
        name: "MyTargetTests",
        dependencies: ["MyTarget"]
    ),
]
```

测试 Target 源文件放在 `Tests/` 目录下，需要声明对被测试 Target 的依赖。

### 4.2 Product（产品）

Product 定义了软件包可供其他软件包或应用使用的**库和可执行文件**。Target 定义内部构建单元，Product 定义对外暴露的接口。没有声明 Product 的 Target 只能在软件包内部使用。

**定义 Library Product：**

```swift
products: [
    .library(
        name: "MyProduct",
        targets: ["MyTarget"]
    ),
],
```

**库的类型：**

| 类型 | 说明 |
|------|------|
| **静态库**（默认） | 编译时链接，有更好的编译优化和更简单的部署方式 |
| **动态库** | 运行时加载，适用于插件系统或进程间共享代码 |

```swift
// 动态库示例
.library(
    name: "LegacyMenuDownloader",
    type: .dynamic,
    targets: ["LegacyMenuDownloader"]
),
```

**可执行文件 Product：**

```swift
products: [
    .executable(name: "MyTool", targets: ["MyTool"]),
],
```

可执行文件通常用于命令行工具，Xcode 会自动生成 `main.swift` 作为入口点。

---

## 五、依赖管理 `[14:00 - 18:00]`

### 5.1 添加依赖

依赖在清单文件的 `dependencies` 参数中声明：

```swift
dependencies: [
    .package(url: "https://github.com/jpsim/Yams", .upToNextMajor(from: "2.0.0")),
],
```

### 5.2 版本要求选项

| 版本要求方式 | 含义 | 示例 |
|------------|------|------|
| `.upToNextMajor(from:)` | 接受指定版本到下一个大版本（不含） | `.upToNextMajor(from: "2.0.0")` 接受 [2.0.0, 3.0.0) |
| `.upToNextMinor(from:)` | 接受指定版本到下一个次版本（不含） | `.upToNextMinor(from: "2.1.0")` 接受 [2.1.0, 2.2.0) |
| `.exact(_:)` | 精确匹配某个版本 | `.exact("2.0.0")` |
| `.branch(_:)` | 使用指定分支的最新提交 | `.branch("main")` |
| `.revision(_:)` | 使用指定的 Git 提交哈希 | `.revision("85cfe06")` |

**重要限制**：`.branch` 和 `.revision` 方式**不允许在已发布的软件包中使用**，公开发布的软件包必须使用语义化版本要求。这两种方式仅适用于本地开发。

### 5.3 依赖解析与锁定

1. SPM 解析所有依赖的版本约束，找到一组兼容版本
2. 解析结果写入 `Package.resolved` 文件（锁定文件），确保团队使用相同版本
3. `Package.resolved` 应提交到版本控制中

```bash
# Xcode 中手动触发依赖解析
# File > Packages > Reset Package Caches
# File > Packages > Resolve Package Versions
```

---

## 六、平台可用性声明 `[18:00 - 19:30]`

### 6.1 声明方式

```swift
let package = Package(
    name: "MyPackage",
    platforms: [
        .macOS(.v10_15),
        .iOS(.v13)
    ],
    // ...
)
```

### 6.2 声明的含义

`platforms` 参数的作用是**告知消费者**本软件包的最低运行环境要求，而**不是限制**只能在声明的平台上使用：

- 声明了 `.iOS(.v13)` 后，依赖此软件包的 app 在 iOS 13 以下编译时 Xcode 会发出警告
- 软件包仍然可以在未声明的平台（如 macOS、tvOS）上使用，前提是代码兼容

---

## 七、发布软件包 `[19:30 - 23:00]`

### 7.1 语义化版本控制（Semantic Versioning）

Apple **强烈推荐**使用语义化版本控制管理软件包版本号：

```
MAJOR.MINOR.PATCH
```

| 版本段 | 何时递增 | 影响 |
|-------|---------|------|
| **MAJOR**（主版本号） | 做了不兼容的 API 变更 | 消费者需要修改代码才能升级 |
| **MINOR**（次版本号） | 做了向后兼容的功能新增 | 消费者无需修改代码即可升级 |
| **PATCH**（修订号） | 做了向后兼容的问题修正 | 无破坏性变更 |

### 7.2 预发布版本标识

```
1.0.0-alpha.1
1.0.0-beta.2
1.0.0-rc.1
```

预发布版本优先级**低于**对应正式版本：`1.0.0-alpha < 1.0.0-beta < 1.0.0`。适合在开发过程中让合作方提前测试新功能。

### 7.3 发布流程

```bash
# 1. 确保所有测试通过
# 2. 创建语义化版本标签
git tag 1.0.0

# 3. 推送标签到远程仓库
git push origin 1.0.0
```

SPM 通过 Git 标签自动发现版本，无需向任何包注册中心注册。Git 即是分发渠道。

---

## 八、编辑软件包 `[23:00 - 25:00]`

### 8.1 远程依赖锁定

通过 SPM 添加的远程依赖**默认不可编辑**，Xcode 以只读模式显示源文件，防止意外修改第三方代码。

### 8.2 本地覆盖远程依赖

当需要调试或修改远程依赖时：

1. 克隆远程依赖仓库到本地
2. 将本地克隆添加为项目中的本地软件包依赖
3. 本地副本将**覆盖**远程依赖，可以自由编辑

**适用场景：**

- **调试**: 怀疑依赖库有 bug 时，添加本地副本进行断点调试
- **贡献上游**: 在本地修改测试后向上游提交 Pull Request
- **定制验证**: 临时添加功能来验证想法

完成编辑后，移除本地副本即可恢复使用远程依赖。

---

## 九、Demo 演示 `[25:00 - 32:00]`

Session 中演示了从零创建开源 Swift 软件包的完整流程：

1. **创建新软件包**: 通过 `File > New > Swift Package` 创建
2. **编写源代码**: 在 `Sources/` 下添加代码，使用 `public` 修饰符定义 API
3. **编写测试**: 在 `Tests/` 下创建测试 Target，使用 XCTest 框架
4. **配置依赖**: 在清单文件中添加第三方依赖和版本约束
5. **发布**: 通过 Git 标签推送到远程仓库

---

## 十、当时的局限性 `[32:00 - 34:00]`

在 WWDC 2019 / Xcode 11 时期，SPM 存在以下限制：

| 限制 | 说明 |
|------|------|
| **仅支持源代码和单元测试** | 不支持图片、素材、Storyboard、XIB 等资源文件 |
| **不支持纯二进制框架** | 无法包含 C/C++ 库的预编译版本 |
| **Playground 集成不完善** | 可导入软件包但体验不够好 |
| **高级功能待完善** | 包注册中心、细粒度权限控制等后来才逐步添加 |

> 这些限制在后续版本中已逐步解决：资源文件支持在 Xcode 12（SPM 5.3）中引入，二进制目标支持也在后续版本添加。

---

## 十一、总结 `[34:00 - End]`

WWDC 2019 Session 410 全面讲解了 Swift Package Manager 的使用方法，从本地软件包的创建到公开发布的完整流程。关键要点如下：

1. **全平台支持**: 从 Xcode 11 开始，SPM 支持所有 Apple 平台，不再限于服务端
2. **本地软件包即子项目**: 即使不公开发布，本地软件包也是组织代码、解耦模块的极佳方式
3. **Package.swift 是核心**: 所有配置都在清单文件中，是唯一的配置真相来源
4. **Target 是构建块，Product 是接口**: Target 定义模块/测试，Product 定义对外暴露的库/可执行文件
5. **语义化版本控制**: 使用 `MAJOR.MINOR.PATCH` 格式，配合预发布标识进行测试
6. **Git 即分发渠道**: 无需注册中心，SPM 直接通过 Git 标签发现和分发软件包

通过正确使用 Swift Package Manager，开发者可以显著提升代码组织能力和项目可维护性。

---

## 时间轴总览

| 章节 | 时间区间 |
|-----|---------|
| 一、为什么要使用 Swift 软件包 | 0:00 - 2:00 |
| 二、创建本地软件包 | 2:00 - 6:00 |
| 三、软件包清单文件（Package.swift） | 6:00 - 10:00 |
| 四、Target 与 Product | 10:00 - 14:00 |
| 五、依赖管理 | 14:00 - 18:00 |
| 六、平台可用性声明 | 18:00 - 19:30 |
| 七、发布软件包 | 19:30 - 23:00 |
| 八、编辑软件包 | 23:00 - 25:00 |
| 九、Demo 演示 | 25:00 - 32:00 |
| 十、当时的局限性 | 32:00 - 34:00 |
| 十一、总结 | 34:00 - End |

---

## 参考资料

- [WWDC 2019 Session 410 视频页面](https://developer.apple.com/cn/videos/play/wwdc2019/410)
- [WWDC 2019 Session 408: 在 Xcode 中使用 Swift 软件包](https://developer.apple.com/videos/play/wwdc2019/408)
- [Apple Developer Documentation - Creating a Standalone Swift Package](https://developer.apple.com/documentation/xcode/creating-a-standalone-swift-package-with-xcode)
- [WWDCNotes 社区笔记](https://wwdcnotes.com/documentation/wwdcnotes/wwdc19-410-creating-swift-packages)
