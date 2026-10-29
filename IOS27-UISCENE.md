# goguma for iOS 27（UIScene 迁移版）

这个仓库是 [emersion/goguma](https://codeberg.org/emersion/goguma)（IRC 客户端，Flutter 项目）
在 `05318d9` 基础上的副本，**唯一改动是补上 iOS 27 必需的 UIScene 迁移**，让它能在 iOS 27 上正常启动。

- 上游版本：goguma 0.9.1（build 1365），Flutter 3.47.8
- 本仓库分支：`main`（= 上游 `05318d9` + 一个迁移提交 + CI）

## 为什么必须改

Apple 从 **Xcode 27（iOS 27 SDK）** 开始强制 UIKit App 使用 `UIScene` 生命周期。
Flutter 官方说明见 [UIScene adoption](https://docs.flutter.dev/release/breaking-changes/uiscenedelegate)：

> Beginning with Xcode 27 (iOS 27 SDK), Apple mandates the UIScene lifecycle for UIKit apps.
> Flutter apps built with Xcode 27 that do not adopt UIScene **fail to launch on startup**.

上游 goguma 尚未迁移（issue [#371 UIScene migration](https://codeberg.org/emersion/goguma/issues/371)，
草稿 PR [#373 WIP: ios: Migrate to UIScene lifecycles](https://codeberg.org/emersion/goguma/pulls/373) 未合并），
所以用 Xcode 27 构建的原版 goguma 在 iOS 27 上一开就闪退。

## 改了什么（只有 2 个文件，照上游 PR #373）

| 文件 | 改动 |
| --- | --- |
| `ios/Runner/AppDelegate.swift` | `AppDelegate` 增加 `FlutterImplicitEngineDelegate`；`GeneratedPluginRegistrant` 注册从 `didFinishLaunchingWithOptions` 移到 `didInitializeImplicitFlutterEngine(_:)` |
| `ios/Runner/Info.plist` | 新增 `UIApplicationSceneManifest`（`UIWindowSceneSessionRoleApplication` → `FlutterSceneDelegate`，`UISceneStoryboardFile = Main`） |

另外新增 `.github/workflows/build-ipa.yml` 用于打未签名 IPA（不属于上游代码）。
Dart 代码、依赖、资源和上游完全一致。

## 构建

### GitHub Actions（推荐）
Actions → `ios-ipa` → Run workflow：
- `runner`: `xcode-27`（Xcode 27.1 / iOS 27 SDK）或 `macos-26`（Xcode 26.x / iOS 26 SDK）
- `flutter_version`: 默认 `3.47.8`
- 产物：`ios_outputs_goguma`（未签名 IPA）、`buildlog_goguma`（构建日志）

push 到 `main` 也会自动构建（只改 `*.md` 不触发）。

### 本地
```bash
flutter pub get
cd ios && pod install && cd ..
flutter build ios --release --no-codesign
# 产物：build/ios/iphoneos/Runner.app，自行打成 Payload/*.ipa
```

## 两种构建产物的区别（Releases 里都有）

| 文件 | 构建环境 | 说明 |
| --- | --- | --- |
| `Goguma_iOS0.9.1_ios27scene_unsigned.ipa` | Xcode 27.1 / iOS 27 SDK | **iOS 27 原生**：带 UIScene 迁移，符合 Apple 强制要求 |
| `Goguma_iOS0.9.1_xcode26_unsigned.ipa` | Xcode 26.6 / iOS 26.5 SDK | **兼容版**：不带 UIScene（走的还是旧生命周期，但用 iOS 26 SDK 构建因此 iOS 27 不强制），与 App Store 版同款 |

两版功能一致；若原生版遇到个别插件兼容问题（推送、分享扩展），用兼容版。

## 安装注意
- bundle id 为上游硬编码的 `me.jeanthomas.goguma`，**与 App Store 版相同**：如已装官方版需先删除，否则签名冲突。
- 未签名 IPA，用 iloader 等工具重签名安装。
- 侧载环境没有 App Group / APNs 权限：**推送通知不可用**，分享扩展可能不生效；IRC 收发、多服务器、TLS、本地数据库正常。

## 与上游同步
```bash
git remote add upstream https://codeberg.org/emersion/goguma.git
git fetch upstream
git rebase upstream/master     # 迁移提交很小，冲突只在 AppDelegate.swift / Info.plist
```
上游 PR #373 合并后，可以直接去掉这里的迁移提交。

## 已知问题
- 迁移补丁是上游 WIP 版本（draft PR），个别依赖旧生命周期的插件（如 `flutter_apns_only`）行为可能与上游不完全一致。
- 深色/浅色、语言、消息收发、bitchat 无关功能均未改动。
