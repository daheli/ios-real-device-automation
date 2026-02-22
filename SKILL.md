---
name: ios-real-device-automation
description: 在 iOS 真机上执行自动化调试与验证，包括日志采集、应用安装卸载、截图取证、进程控制、WDA 启动与端口转发、Deep Link/Universal Link 唤起。用户提出真机联调、URL 跳转验证、WDA 连通性排查或证据留存时使用。
---

# iOS Real Device Automation

使用 `ios`（go-ios CLI）和 `xcrun devicectl` 在真机上完成可复现、可取证的自动化操作。

## 遵循原则

1. 在命令中显式传入设备标识：优先使用 `--udid "<UDID>"` 或 `--device "<UDID>"`。
2. 对 URL 唤起使用 `xcrun devicectl ... --payload-url`，不要依赖 `ios launch --arg` 实现系统级路由。
3. 在关键操作后补充证据：至少保存一张截图或一段可检索日志。
4. 在输出中给出结论和下一步动作，保证结果可复核。

## 按顺序执行工作流

1. 识别目标设备并确认系统信息。
2. 根据任务执行安装、卸载、启动、关闭、截图或日志采集。
3. 如需 UI 自动化，启动 WDA 并转发 8100 端口。
4. 如需路由验证，使用 `devicectl --payload-url` 执行 Deep Link 或 Universal Link。
5. 收集日志关键字与截图，输出结构化结论。

## 使用命令模板

### 设备识别

```bash
ios list --details
ios info --udid "<UDID>"
ios sysinfo --udid "<UDID>"
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

### 安装与卸载

```bash
ios install --path "./YourApp.ipa" --udid "<UDID>"
ios uninstall "com.example.app" --udid "<UDID>"
```

### 进程与截图

```bash
ios launch "com.example.app" --udid "<UDID>"
ios kill "com.example.app" --udid "<UDID>"
ios ps --apps --udid "<UDID>" --nojson
ios screenshot --udid "<UDID>" --output "./shot.png"
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
