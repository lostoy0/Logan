# CLAUDE.md

本文件用于指导 Claude Code（claude.ai/code）在本仓库中协作。

## 语言约定（Language Conventions）

本项目的文档、代码注释、AI 对话默认以**中文**为主，仅在以下场景使用英文：

- **代码标识符**：变量名、函数名、类名、模块名、文件名等保持英文，不要为追求"中文化"而改名。
- **专业术语 / 技术专有名词**：例如 `mmap`、`AES`、`JNI`、`MethodChannel`、`SwiftPM` 等。如有公认的中文译名，采用"中文（English）"对照形式，例如"内存映射（mmap）"、"页大小（page size）"；若中文译名生硬或不通用，则保留英文原词。
- **命令行命令、API 名称、错误日志原文、第三方文档引用**：保留英文。
- **Git 提交信息（commit message）/ PR 标题与正文 / 代码注释（comments / docstrings）**：默认中文，必要的英文术语按上述规则对照。

仅在中文表达明显不合适或会引起歧义时才使用英文。

## 仓库结构（Repository Shape）

x-logan 是一个基于 Logan 的跨平台日志系统。本仓库是包含五个松耦合组件与示例工程的 monorepo（单体仓库），**没有**顶层统一构建——每个组件都有各自的工具链。`Logan` 作为现有 SDK、API、模块和发布包的兼容名称保留。

```
x-logan/
├── Logan/Clogan/         # 共享 C 内核（mmap + AES + cJSON + zlib）。被 iOS、macOS、Android（通过 JNI）和 Flutter 共用。
├── Logan/mbedtls/        # 内置（vendored）的 mbedTLS 源码，编译进 Clogan，用于 AES 加密。
├── Logan/iOS/            # 包裹 Clogan 的 Objective-C 薄封装（Logan.h/.m）。以 `Logan` CocoaPod / SwiftPM target 形式发布。
├── Logan/WebSDK/         # TypeScript 浏览器 SDK（基于 IndexedDB，独立于 C 内核）。在 npm 上发布为 `logan-web`。
├── Logan/Server/         # Java 8 + Spring MVC + MyBatis 后端。负责接收上传的日志、解密并写入 MySQL。
├── Logan/LoganSite/      # React 16 + Redux + antd 前端，访问 Server。基于 CRA 脚手架并已 eject。
├── Logan/scripts/migration/mysql/  # 提供给 docker-compose 使用的 golang-migrate SQL 迁移脚本。
├── Flutter/              # Flutter 插件（`flutter_logan`），将 Dart 桥接到原生 iOS/Android Logan。
├── Example/
│   ├── Logan-Android/    # Android Studio 工程。包含可发布的 `logan` 库模块（com.dianping.logan）和演示 app。
│   ├── Logan-iOS/        # 使用本地 podspec 的 CocoaPods 示例工程。
│   ├── Logan-macOS/      # Xcode 的 macOS 演示工程。
│   ├── Logan-NodeServer/ # 为 Web SDK 演示提供接收端测试用的极简 TypeScript 服务。
│   └── Logan-WebSDK/     # Web SDK 的演示工程。
├── Logan.podspec         # CocoaPods 规范文件——拉取 Logan/iOS + Logan/Clogan + Logan/mbedtls。
├── Package.swift         # SwiftPM 清单——三个 target：mbedtls → CLogan → Logan。
└── Logan/docker-compose.yaml + Logan/deploy.sh   # 一键启动的本地全栈：MySQL + Server + LoganSite + phpMyAdmin。
```

Android 库模块的物理位置在 `Example/Logan-Android/logan/`（**不**在 `Logan/` 下）。其 `CMakeLists.txt` 会向上引用 `Logan/Clogan` 来拉取共享 C 源码——因此当修改 C 内核时，iOS Pod 构建和 Android NDK 构建会从同一份源文件中同步生效。

## 跨文件的架构要点（Cross-file Architecture Notes）

**一份 C 内核，三套 FFI 接口面（FFI surfaces）。** `Logan/Clogan/clogan_core.{c,h}` 是端侧日志流水线的唯一源（single source of truth）。对外 API 只有 5 个函数：`clogan_init`、`clogan_open`、`clogan_write`、`clogan_flush`、`clogan_debug`（见 `clogan_core.h:55-87`）。日志先写入 mmap（内存映射）缓存，再使用 AES-128（mbedTLS）加密并以 gzip（zlib）压缩，按天落盘为单独文件。各原生封装层最终都会汇聚到这份 C 内核：

- iOS / macOS：`Logan/iOS/Logan.m` 直接调用 `clogan_*`。
- Android：`Example/Logan-Android/logan/src/main/jni/clogan_protocol.c` 是 JNI 桥接层，由 `CLoganProtocol.java` 调用。
- Flutter：`Flutter/lib/flutter_logan.dart` → MethodChannel → 原生 iOS/Android Logan。

修改 C 接口签名时，**三套桥接层必须同步更新**。

**线程模型（Android）。** 所有写入都通过 `LoganControlCenter`（`Example/Logan-Android/logan/src/main/java/com/dianping/logan/LoganControlCenter.java`）入队到 `ConcurrentLinkedQueue<LoganModel>`，由单一线程 `LoganThread` 消费。**C 内核本身并不是线程安全的**，依赖该"单写者"不变量来保证正确性。头文件中明确写到："前4个接口在Thread中调用"。**不要**在任意线程中调用 `clogan_write`。

**Web SDK 与 C 内核相互独立。** `Logan/WebSDK` 与 C 内核**不共享代码**。它使用 IndexedDB（通过 `idb-managed`）做存储，用 `crypto-js` 做 AES，但上传到 Server 的报文（wire format）与服务端是兼容的。每日存储上限约 7MB，保留 7 天（参见 `Logan/WebSDK/README.md` 中的 "Limits" 一节）。

**Server 是 WAR，不是 Spring Boot 应用。** `Logan/Server/pom.xml` 打包出一个可部署到 Tomcat 的 WAR（`<packaging>war</packaging>`）。其 Dockerfile 用 Maven 构建后，把 WAR 投放到 `tomcat:7.0.61-jre8` 镜像中。数据库配置位于 `Server/src/main/resources/db.properties`，使用占位符（`host`、`port`、`database`、`=username`、`=password`），由 `deploy.sh` 在部署时通过 `sed` 替换。

**16KB 页大小兼容（Android NDK）。** 近期提交（参见 `git log`）围绕 Android 15 的 16KB 页大小要求展开。`Example/Logan-Android/logan/CMakeLists.txt:30` 增加了 `-Wl,-z,max-page-size=16384`。NDK 版本在 `logan/build.gradle:10` 中固定为 `23.2.8568313`。若要改动 NDK 构建，请先阅读相关提交信息——其中记录了在更新版 NDK 与 Gradle/AGP 兼容性之间的取舍历史。

## 常用命令（Common Commands）

### iOS / macOS（C 内核 + Obj-C 封装）

```bash
# 构建 iOS 示例 app
cd Example/Logan-iOS && pod install && open Logan-iOS.xcworkspace

# 独立构建 C 内核（产出用于本地测试的 clogan_runner 可执行文件）
cd Logan && cmake -S Clogan -B Clogan/build && cmake --build Clogan/build

# 在仓库根目录用 SwiftPM 构建
swift build
```

仓库根目录的 `Logan.podspec` 引用了 `Logan/iOS/*.{h,m}` 与 `Logan/Clogan/*.{h,c}`——新增的 C 文件请保持放在 `Logan/Clogan/` 下，以便 Pod 与 SwiftPM 两条接入路径都能识别。

### Android（库 + 演示 app）

```bash
cd Example/Logan-Android

# 全量构建（需要通过 SDK Manager 安装 NDK 23.2.8568313）
./gradlew assembleDebug

# 仅构建库（AAR）
./gradlew :logan:assembleRelease

# 单元测试（`logan` 模块使用 junit:4.12）
./gradlew :logan:testDebugUnitTest

# 单个测试类
./gradlew :logan:testDebugUnitTest --tests com.dianping.logan.SomeTest

# 设备上的仪器化测试（instrumented tests）
./gradlew :logan:connectedDebugAndroidTest
```

工具链：AGP 7.4.2、Gradle 7.5（参见 `Example/Logan-Android/build.gradle:13` 与 `gradle/wrapper/gradle-wrapper.properties`），compileSdk 34，minSdk 21，NDK 23.2.8568313。

### Web SDK

```bash
cd Logan/WebSDK
npm install
npm test                  # jest --coverage
npx jest path/to/file.test.ts   # 单个测试文件
npm run build             # tsc → build/
npm run demo              # 构建并用 webpack 打包演示
npm run start:dev         # 启动 webpack-dev-server，实时调试演示
```

### Server（Java）

```bash
cd Logan/Server
mvn clean package         # 产出 target/logan-web-beta-1.0-SNAPSHOT.war
mvn test                  # JUnit 4.12 测试
mvn test -Dtest=SomeTest#someMethod
```

### LoganSite（React 前端）

```bash
cd Logan/LoganSite
# 按 README 要求：Node 10.15.3 / yarn 1.15.2
yarn          # 或 npm install
yarn start    # 启动开发服务器，需在 .env.development 中配置 API_BASE_URL
yarn build
yarn test     # CRA 风格的 jest，默认 --watch
yarn test src/path/to/file.test.js  # 单个测试文件
```

首次配置时，将 `.env.development.example` 复制为 `.env.development`，并把 `API_BASE_URL` 设置为 Server 的访问地址。

### 全栈（Docker）

```bash
cd Logan
./deploy.sh   # 改写 db.properties + .env，并执行 docker-compose up -d --build
```

会拉起：后端（`:8888`）、MySQL 8（`:23306`）、phpMyAdmin（`:10050`）、LoganSite 前端（`:3000`），以及一个一次性运行的 `db-migrate` 容器——它通过 `migrate/migrate` 执行 `Logan/scripts/migration/mysql/*` 中的迁移脚本。

### Flutter 插件

```bash
cd Flutter
flutter pub get
cd example && flutter run    # 在已连接的设备上运行示例 app
flutter test                 # 插件自身的测试位于 Flutter/test/
```

## 约定（Conventions）

- **版权头（copyright header）**：每个新建源文件**必须**带上版权头，原文位于 `CONTRIBUTING.md:23-45`（MIT 协议，"Copyright (c) 2018-present, 美团点评"）。现有的 C / Obj-C / Java / Swift 文件都已包含该头。
- **PR 目标分支为 `master`**。推送前请将提交压缩（squash），参见 `CONTRIBUTING.md`。
- **AES 密钥 / IV 在每一层都是 16 字节**（iOS 的 `loganInit`、Android 的 `LoganConfig.setEncryptKey16/setEncryptIV16`、C 的 `clogan_init` 中的 `encrypt_key16`/`encrypt_iv16`）。客户端与服务端的 key/IV 不一致会导致日志无法解密。
- **日志类型 `1` 是 Logan 在 Android 内部保留的**（参见 `Example/Logan-Android/README.md:39`）。业务代码的类型值应从 2 开始。
- **自定义的 `SendLogRunnable.sendLog()` 实现中必须调用 `finish()`**——否则 Android 端的发送队列会卡住。
