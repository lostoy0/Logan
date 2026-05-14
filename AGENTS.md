# Repository Guidelines

## 项目结构与模块组织

x-logan 是多平台 monorepo，没有统一的顶层构建入口；`Logan` 仍作为 SDK/API 兼容名称保留。`Logan/Clogan/` 是共享 C 核心（shared C core），`Logan/mbedtls/` 是内置加密依赖，`Logan/iOS/` 通过 `Logan.podspec` 和 `Package.swift` 提供 CocoaPods / SwiftPM 封装。Android 库位于 `Example/Logan-Android/logan/`，示例项目集中在 `Example/`。`Flutter/` 是 Flutter 插件，`Logan/WebSDK/` 是浏览器 SDK（Web SDK），`Logan/Server/` 是 Java 后端，`Logan/LoganSite/` 是 React 管理端。测试随组件放置，例如 `Logan/WebSDK/tests/`、`Flutter/test/` 和各平台示例测试目录。

## 构建、测试与开发命令

从目标组件目录运行对应命令：

- `swift build`：在仓库根目录构建 SwiftPM targets。
- `cd Logan && cmake -S Clogan -B Clogan/build && cmake --build Clogan/build`：构建 C 核心 runner。
- `cd Example/Logan-Android && ./gradlew assembleDebug`：构建 Android 示例和库。
- `cd Example/Logan-Android && ./gradlew :logan:testDebugUnitTest`：运行 Android 库单元测试。
- `cd Logan/WebSDK && npm install && npm test && npm run build`：安装依赖、运行 Jest coverage、构建 TypeScript。
- `cd Logan/Server && mvn test && mvn clean package`：测试并打包 WAR。
- `cd Logan/LoganSite && yarn && yarn test && yarn build`：安装、测试并构建 React 站点。
- `cd Flutter && flutter pub get && flutter test`：校验 Flutter 插件。

## 语言与写作规范

项目文档、代码注释、提交说明和 AI 协作交流以中文为主。只有代码、文件名、命令、API、配置键、协议名、框架名或行业通用术语必须使用英文时才保留英文。专有名词首次出现时尽量中英文对照，例如“共享 C 核心（shared C core）”“浏览器 SDK（Web SDK）”。避免无必要的全英文说明。

## 代码风格与命名约定

遵循被修改组件的既有风格。C、Objective-C、Java、Swift 新源码文件需加入 `CONTRIBUTING.md` 中的 MIT copyright header。公共 API 变更必须同步考虑 C 核心、iOS wrapper、Android JNI bridge 和 Flutter bridge。TypeScript 遵循 `Logan/WebSDK/tsconfig.json`、Jest 和 ESLint；React 代码遵循 CRA `react-app` ESLint 配置。

## 测试规范

在变更所属组件内新增或更新测试。Web SDK 使用 `Logan/WebSDK/tests/*.test.ts`，Flutter 使用 `Flutter/test/`，Server 使用 `Logan/Server/src/test/` 下的 Maven/JUnit 测试，Android 使用 Gradle 单元测试或仪器测试。修改 C 核心时，至少运行 C/SwiftPM 构建和受影响平台测试。

## 提交与 Pull Request 规范

历史提交常用 `feat:`、`fix:` 前缀；保持提交范围清晰、描述具体。PR 面向 `master`，提交前格式化代码、确认相关构建可通过，并按需 squash commits。PR 描述需说明影响平台、关键变更、关联 issue；涉及 UI、构建或运行行为时附截图、日志或复现信息。
