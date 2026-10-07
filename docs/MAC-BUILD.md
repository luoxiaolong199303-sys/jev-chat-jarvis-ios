# Mac 端编译与真机运行指南（缓存全部落在 D 盘）

> 适用场景：本机为 Windows + macOS 双系统（Boot Camp），Windows 的 **D 盘**在 macOS 下挂载为 `/Volumes/D`。
> 目标：在 Mac 上编译 `jev-chat-jarvis-ios`，**构建缓存（DerivedData / Build / Precompiled）与运行时缓存全部写入 `/Volumes/D/jev-ios-cache`，不写 C 盘（macOS 上即系统盘 `~/Library/Developer/Xcode/DerivedData`）**。

---

## 0. 前置条件（在 Mac 上）

- macOS + Xcode（建议 Xcode 15/16，App Store 安装）
- Xcode Command Line Tools：`xcode-select --install`
- [XcodeGen](https://github.com/yonaskolb/XcodeGen)（用 `project.yml` 生成 `.xcodeproj`）：
  ```bash
  brew install xcodegen
  # 或： mint install yonaskolb/XcodeGen
  ```
- D 盘已挂载并可写：
  ```bash
  ls /Volumes/D          # 应能看到你的 Windows D 卷
  touch /Volumes/D/.writetest && echo "D 盘可写" && rm /Volumes/D/.writetest
  ```
- Apple ID（免费即可跑模拟器；真机签名 + App Group 需付费 $99 开发者账号，见第 6 节说明）
- 仓库已克隆到 Mac 本地（从你自己的 fork 拉取）：
  ```bash
  git clone https://github.com/luoxiaolong199303-sys/jev-chat-jarvis-ios.git
  cd jev-chat-jarvis-ios
  git checkout feat/ios-cache-d-drive
  ```

---

## 1. 生成 Xcode 工程

`project.yml` 已内置缓存路径重定向，直接生成即可：

```bash
xcodegen generate      # 读取 project.yml → 产出 JevJarvis.xcodeproj
```

生成后可用 `open JevJarvis.xcodeproj` 在 Xcode 中打开。

---

## 2. 预建缓存目录（可选，build.sh 会自动建）

```bash
mkdir -p /Volumes/D/jev-ios-cache/{DerivedData,Build/Products,Build/Intermediates,Build/Precompiled,runtime,logs}
```

如需临时改路径，传入环境变量（代码中 `JevPaths.cacheRoot` 会优先读它）：

```bash
export JEV_CACHE_ROOT=/Volumes/D/jev-ios-cache
```

---

## 3. 编译：模拟器版（最常用）

### 方式 A：用自带脚本（推荐）

```bash
./build.sh sim          # Debug + iphonesimulator，缓存全部走 D 盘
```

### 方式 B：裸 xcodebuild（便于排查）

```bash
xcodebuild -project JevJarvis.xcodeproj \
  -scheme JevJarvis \
  -configuration Debug \
  -sdk iphonesimulator \
  -derivedDataPath /Volumes/D/jev-ios-cache/DerivedData \
  SYMROOT=/Volumes/D/jev-ios-cache/Build/Products \
  OBJROOT=/Volumes/D/jev-ios-cache/Build/Intermediates \
  SHARED_PRECOMPS_DIR=/Volumes/D/jev-ios-cache/Build/Precompiled
```

---

## 4. 运行：模拟器

```bash
# 在 Xcode 里选模拟器设备后 ⏵ Run 即可；
# 或用命令行拉起：
xcrun simctl boot "iPhone 16"            # 设备名按你装的模拟器调整
xcrun simctl install booted \
  /Volumes/D/jev-ios-cache/Build/Products/Debug-iphonesimulator/JevJarvis.app
xcrun simctl launch booted com.jevchat.jarvis
```

键盘扩展（Keyboard Extension）需先在模拟器的「设置 → 键盘 → 添加新键盘」里启用，再切到 App 内测试。

---

## 5. 编译：真机（sideload / 开发签名）

```bash
./build.sh device        # Release + iphoneos，需提前在 Xcode 配好 Team
```

或裸命令（把 `YOUR_TEAM_ID` 换成你的开发者团队 ID）：

```bash
xcodebuild -project JevJarvis.xcodeproj \
  -scheme JevJarvis \
  -configuration Release \
  -sdk iphoneos \
  -allowProvisioningUpdates \
  DEVELOPMENT_TEAM=YOUR_TEAM_ID \
  -derivedDataPath /Volumes/D/jev-ios-cache/DerivedData \
  SYMROOT=/Volumes/D/jev-ios-cache/Build/Products \
  OBJROOT=/Volumes/D/jev-ios-cache/Build/Intermediates \
  SHARED_PRECOMPS_DIR=/Volumes/D/jev-ios-cache/Build/Precompiled
```

Xcode 首次真机运行会在 **Signing & Capabilities** 自动生成开发证书与 Provisioning Profile（免费账号可用，7 天有效期，需重签）。

---

## 6. 出 IPA（归档 + 导出）

```bash
./build.sh archive       # 产出 .xcarchive 到 D 盘缓存目录
```

再准备 `ExportOptions.plist`（按你的分发方式选 `development` / `ad-hoc` / `app-store`）：

```bash
xcodebuild -exportArchive \
  -archivePath /Volumes/D/jev-ios-cache/Build/Products/JevJarvis.xcarchive \
  -exportPath /Volumes/D/jev-ios-cache/Build/IPA \
  -exportOptionsPlist ExportOptions.plist
```

> 真机 / App Group 说明：本项目 App 与键盘扩展共用 App Group `group.com.jevchat.jarvis`（用于共享配置与密钥）。**App Group 能力在免费账号下不可用**，需 $99 付费开发者账号才能在 Signing & Capabilities 中勾选并生效；否则键盘扩展无法读取 App 侧写入的共享数据。构建缓存与 DerivedData 不受影响，始终在 D 盘。

---

## 7. 校验：缓存确实在 D 盘、不在系统盘

```bash
# 构建缓存应在 D 盘
ls -d /Volumes/D/jev-ios-cache/DerivedData && echo "✅ DerivedData 在 D 盘"

# 系统盘（macOS 的 C 盘等价物）不应出现本项目的 DerivedData
ls ~/Library/Developer/Xcode/DerivedData 2>/dev/null | grep -i jev \
  && echo "⚠️ 发现 C 盘残留，检查 project.yml / build.sh" \
  || echo "✅ 系统盘无本项目 DerivedData"

# 运行时缓存（仅当 JEV_USE_DISK_CACHE=1 时启用磁盘缓存）
JEV_USE_DISK_CACHE=1 ./build.sh sim
ls -d /Volumes/D/jev-ios-cache/runtime && echo "✅ 运行时缓存在 D 盘"
```

> **设备端限制（已在 `docs/CACHE.md` 说明）**：真机运行时，App 沙盒内的 `Library/Caches` 与 App Group 存储**受 iOS 沙盒限制无法重定向到外部卷**，这部分仍落在设备本机。可重定向到 D 盘的是：Xcode 构建缓存（DerivedData / SYMROOT / OBJROOT / SHARED_PRECOMPS_DIR）以及 Mac 模拟器运行时缓存。代码已通过 `JevPaths` 显式设定路径并在 DEBUG 下打印，确保构建侧零写入系统盘。

---

## 8. 诊断日志

- DEBUG 构建启动时会在 Xcode 控制台打印 `JevPaths.dump()`（缓存根、各子目录绝对路径）。
- Mac 上会把诊断追加写入 `/Volumes/D/jev-ios-cache/logs/jev-diag.log`（真机因沙盒静默失败，符合预期）。

---

## 9. 常用命令速查

| 目的 | 命令 |
|------|------|
| 生成工程 | `xcodegen generate` |
| 模拟器编译+运行 | `./build.sh sim` |
| 真机编译 | `./build.sh device` |
| 归档出包 | `./build.sh archive` |
| 打印当前缓存路径配置 | `./build.sh paths` |
| 查看缓存目录占用 | `du -sh /Volumes/D/jev-ios-cache` |
