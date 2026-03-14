---
name: ios-real-device-automation
description: 在 iOS 真机上执行自动化调试与验证，包括主工程编译（xcodebuild）、应用安装（devicectl）与拉起（ios/devicectl）、日志采集(idevicesyslog)、截图取证、进程控制、WDA 启动与端口转发、Deep Link/Universal Link 唤起。用户提出真机联调、编译安装运行、URL 跳转验证、WDA 连通性排查或证据留存时使用。
---

# iOS Real Device Automation

使用 `ios`（go-ios CLI）和 `xcrun devicectl` 在真机上完成可复现、可取证的自动化操作。

## 遵循原则

1. 在命令中显式传入设备标识：优先使用 `--udid "<UDID>"` 或 `--device "<UDID>"`。
2. 对 URL 唤起使用 `xcrun devicectl ... --payload-url`，不要依赖 `ios launch --arg` 实现系统级路由。
3. 安装优先使用 `xcrun devicectl device install app`（稳定性优先）；执行 `ios launch` 前先确保 tunnel 可用：临时执行 `ios tunnel start`，或使用 `nohup` 后台常驻。
4. 已有可用构建产物（`DerivedData/.../Debug-iphoneos/*.app`）时，优先直接安装，避免重复编译。
5. `xcodebuild` 默认使用 `/usr/bin/time -p` 输出总耗时（`real/user/sys`），便于评估构建性能。
6. 非明确要求时不要执行 `clean`；`clean` 会显著增加后续编译耗时，仅在排查缓存或用户明确要求时使用。
7. 在关键操作后补充证据：至少保存一张截图或一段可检索日志。
8. 在输出中给出结论和下一步动作，保证结果可复核。

## 按顺序执行工作流

1. 识别目标设备并确认系统信息。
2. 建立或校验 tunnel（推荐常驻后台 tunnel）。
3. 编译主工程（默认带总耗时输出，Debug + 真机 destination），或复用已有 `.app` 构建产物。
4. 安装应用到真机（`xcrun devicectl device install app`）。
5. 拉起应用（`ios launch --kill-existing`）。
6. 如需 UI 自动化，启动 WDA 并转发 8100 端口。
7. 如需路由验证，使用 `devicectl --payload-url` 执行 Deep Link 或 Universal Link。
8. 收集日志关键字与截图，输出结构化结论。

## 使用命令模板

### 设备识别

```bash
ios list
ios info --udid "<UDID>"
ios info lockdown --udid "<UDID>"
```

### Tunnel（安装/拉起前置）

```bash
# 一次性启动 tunnel
ios tunnel start --userspace --udid "<UDID>" --nojson

# 推荐：后台常驻，避免每次重复执行
nohup ios tunnel start --userspace --udid "<UDID>" --nojson > /tmp/ios-tunnel.log 2>&1 &

# 校验 tunnel 状态
ios tunnel ls --udid "<UDID>" --nojson
```

### 编译主工程（Debug + 真机）

```bash
# 默认：总耗时输出（real/user/sys）, 并且明确告诉用户编译时间
/usr/bin/time -p xcodebuild \
  -quiet \
  -workspace "<WORKSPACE_PATH>" \
  -scheme "<SCHEME>" \
  -configuration Debug \
  -destination 'generic/platform=iOS' \
  build
```

### 可选：Clean + Build（仅在明确要求时使用）

```bash
/usr/bin/time -p xcodebuild \
  -quiet \
  -workspace "<WORKSPACE_PATH>" \
  -scheme "<SCHEME>" \
  -configuration Debug \
  -destination 'generic/platform=iOS' \
  clean build
```

说明：`clean` 会使全量重编译，显著增加耗时。除非用户明确要求或需要排查缓存问题，否则不执行。

注意: 若项目根目录提供 `buildServer.json`，优先使用其中的 `workspace`, `build_root` 与 `scheme`。

### 复用已有构建产物

```bash
# 常见产物目录
ls "<DERIVED_DATA>/Build/Products/Debug-iphoneos"

# 目标 app 示例
<DERIVED_DATA>/Build/Products/Debug-iphoneos/<APP_NAME>.app
```

### 日志采集（仅使用 `idevicesyslog`）

`idevicesyslog` 完全替代 `ios syslog`。原因：`ios syslog` 在中文日志场景下存在可读性问题。

进程名在记录、分享和沉淀文档时必须脱敏，统一使用占位符 `<APP_PROCESS>`。

```bash
# 实时看主进程日志（只保留 <APP_PROCESS>[pid]）
idevicesyslog -u "<UDID>" | rg --line-buffered '<APP_PROCESS>\[[0-9]+\]'

# 主进程中仅看错误/告警
idevicesyslog -u "<UDID>" | rg --line-buffered '<APP_PROCESS>\[[0-9]+\].*(<Error>|<Fault>|<Warning>)'

# 一边查看一边落盘
idevicesyslog -u "<UDID>" | rg --line-buffered '<APP_PROCESS>\[[0-9]+\]' | tee /tmp/app-main.log

# 后台持续采集
nohup idevicesyslog -u "<UDID>" | rg --line-buffered '<APP_PROCESS>\[[0-9]+\]' >> /tmp/app-main.log 2>/tmp/app-main.err &
echo $! > /tmp/app-main.pid

# 停止后台采集
kill "$(cat /tmp/app-main.pid)"
```

先按 `<APP_PROCESS>\[[0-9]+\]` 缩小范围，再叠加业务关键词（如 `HTTPDNS|ListHelper|crash|error`）定位问题。

### 多行日志过滤（关键词命中但需要展示后续多行）

`rg "<搜索词>"` 只会输出命中关键词的单行；当日志是多行结构（如对象/JSON）时，需使用按块过滤来保留后续内容：

```bash
idevicesyslog -u "<UDID>" | awk '
/<APP_PROCESS>\[[0-9]+\].*<搜索词>/ { print; in_block=1; next }
in_block {
  print
  if ($0 ~ /^\}/) in_block=0
}
'
```

### 安装与卸载

```bash
xcrun devicectl device install app --device "<UDID>" "./YourApp.ipa"
xcrun devicectl device install app --device "<UDID>" "<DERIVED_DATA>/Build/Products/Debug-iphoneos/<APP_NAME>.app"
ios uninstall "com.example.app" --udid "<UDID>"
```

### 进程与截图

```bash
ios launch "com.example.app" --udid "<UDID>" --kill-existing --nojson
ios kill "com.example.app" --udid "<UDID>"
ios ps --apps --udid "<UDID>" --nojson
ios screenshot --udid "<UDID>" --output "./shot.png"
```

### 编译 + 安装 + 拉起（一键模板）

```bash
# 1) 编译
/usr/bin/time -p xcodebuild -quiet \
  -workspace "<WORKSPACE_PATH>" \
  -scheme "<SCHEME>" \
  -configuration Debug \
  -destination 'generic/platform=iOS' \
  build

# 2) 安装（优先复用已有构建产物）
xcrun devicectl device install app --device "<UDID>" "<DERIVED_DATA>/Build/Products/Debug-iphoneos/<APP_NAME>.app"

# 3) 拉起
ios launch "<BUNDLE_ID>" --udid "<UDID>" --kill-existing --nojson
```

### WDA 启动与端口转发

```bash
ios runwda \
  --bundleid "<WDA_BUNDLE_ID>" \
  --testrunnerbundleid "<WDA_RUNNER_BUNDLE_ID>" \
  --xctestconfig "<WDA_XCTESTCONFIG>" \
  --udid "<UDID>"

ios forward --udid "<UDID>" 8100 8100
```

可选：

```bash
ios wdaproxy --udid "<UDID>"
```

## 执行 URL 唤起验证

### Deep Link（scheme）

```bash
xcrun devicectl device process launch \
  --device "<UDID>" \
  --payload-url "demo://question/49242761" \
  com.demo.ios-dev
```

### Universal Link（https）

```bash
xcrun devicectl device process launch \
  --device "<UDID>" \
  --payload-url "https://www.demo.com/question/49242761" \
  com.demo.ios-dev
```

### 冷启动路由验证

```bash
xcrun devicectl device process launch \
  --device "<UDID>" \
  --terminate-existing \
  --payload-url "demo://question/49242761" \
  com.demo.ios-dev
```

注意：`devicectl` 使用 `--payload-url`；`ios launch` 不提供等价的系统级 URL 唤起参数。

## 输出结果模板

按以下字段返回执行结果：

1. 设备信息：`name`、`udid`、`iOS version`
2. 执行命令：脱敏后的关键命令
3. 结论：成功或失败
4. 证据：截图路径与日志路径
5. 下一步：失败时的排查建议

## 优先排查项

- 只拉起未跳页：改用 `devicectl --payload-url`，不要使用 `ios launch --arg` 作为跳页手段。
- Universal Link 不回跳：核对 `Associated Domains`、AASA 和目标包安装状态。
- WDA 不可用：先确认签名与 Runner 安装，再检查 `ios forward --udid "<UDID>" 8100 8100`。
- `ios launch` 报 `InvalidService`：先检查 tunnel 和开发镜像，再执行拉起。
