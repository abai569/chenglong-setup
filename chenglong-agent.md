# 橙龙桌面 · 安装 / 设为主屏幕 说明（给 Agent 读）

这份文档写给能在用户电脑上运行命令的 AI。目标是：**把橙龙桌面装到用户的电视上，或把它设成主屏幕，并确认按主页键真的回到它。**

电视是 Android 盒子/电视，包名 `app.orchard.tv`。全程不需要用户去电视设置里翻菜单——能用命令做的都用命令做，必须用户在电视上按遥控器的地方，明确说出来。

---

## 0. 先弄清楚电视上现在有什么

按顺序问用户，或者直接试：

1. 电视上装了 **橙龙输入法**（`app.orchard.ime`）？→ 走通道 A。
2. 电视上装了 **橙龙桌面**（`app.orchard.tv`）并且正在运行？→ 走通道 B。
3. 能连 **ADB**（电视开了网络调试）？→ 走通道 C。
4. 都没有（全新电视）→ 只能 U 盘，见通道 D。

通道 A 最快、最不挑机器；通道 D 最慢但是唯一一定能成的。

---

## 1. 通道 A：橙龙输入法 9988 端口（首选）

电视上装了橙龙输入法时，它自带一个 HTTP 接收端在 **9988**：

```bash
curl -F "file=@橙龙桌面-1.4.8-S.apk" -F "autoInstall=true" http://<TV-IP>:9988/upload
```

`autoInstall=true` 会让电视自己拉起系统安装器，用户只需要在电视上按「安装」。请求返回 200 不代表装好了，**必须按第 4 节验收**。

## 2. 通道 B：橙龙桌面 8899 端口（已装桌面时）

橙龙桌面运行时在 **8899** 提供同样的接收端，先传包再拉起安装器：

```bash
curl -F "file=@橙龙桌面-1.4.8-S.apk" http://<TV-IP>:8899/api/files/upload
curl -X POST http://<TV-IP>:8899/api/files/open
```

传包返回的 JSON 里有落盘路径；`/api/files/open` 会让电视用 FileProvider + `ACTION_VIEW` 拉起系统安装器。这条不需要 ADB，但要求电视上桌面正在运行（探活：`curl -s -m 3 http://<TV-IP>:8899/` 有响应）。

## 3. 通道 C：ADB

### 环境约定（必须遵守）

- adb 全路径：`I:\Program Files (x86)\Android\Sdk\platform-tools\adb.exe`
- 服务端口 **固定 5061**：`set ANDROID_ADB_SERVER_PORT=5061`
- **不许 `adb kill-server`**，**不许自己 `start-server`**。用户机器上 5061 有一个常驻 daemon，杀了它所有会话一起断，默认 5037 上只有幽灵 offline 设备。
- 5555 连不上时不要反复重试，先让用户看盒子/电视的「网络调试」是否开着。

```bash
set ANDROID_ADB_SERVER_PORT=5061
adb connect <TV-IP>:5555
adb devices
adb -s <TV-IP>:5555 install -r "橙龙桌面-1.4.8-S.apk"
```

装不上时的错误分三类：

| 报错 | 含义 | 怎么办 |
| --- | --- | --- |
| `INSTALL_FAILED_VERSION_DOWNGRADE` | 包里的 versionCode 比电视上的低 | 换更新的包，或先卸载旧版（会丢设置） |
| `INSTALL_FAILED_INCONSISTENT_CERTIFICATES` | 不是同一个签名 | 只能卸载重装，先提醒用户会丢设置 |
| `INSTALL_FAILED_INSUFFICIENT_STORAGE` | 空间不够 | 清理电视存储 |

TCL 一类固件把 `install_switch_flag` 关死时，`pm install` 会被系统拒掉，这时不要硬试，改用通道 A/B，或让用户 U 盘装。

## 4. 设成主屏幕（接管 Home）

装好之后有两种接管方式，S 版都能用，G 版只有第一种：

1. **桌面守护（无障碍）** —— 打开橙龙桌面 → 控制中心 → 「打开桌面守护」→ 系统无障碍页里打开「橙龙桌面守护」。从此按 Home 键由守护把前台拉回桌面。这是 G 版唯一的接管方式，TCL 等装不上替身的机器也靠它。
2. **主页替身** —— S 版可选，一个声明了 HOME 的独立包（`com.ts.launcher`）。TCL 多数固件装不上；装不上就只留守护，不要反复试。

能连 ADB 时，守护可以用广播配置，不必让用户点：

```bash
adb -s <TV-IP>:5555 shell am broadcast -a app.orchard.tv.CONFIG \
  --es guard true --es filter_home true
```

成功与否要看返回里的 `resultData data=`，不是 `result=0`。回滚用 `--es reset true`。

## 5. 验收（必做，把原始输出贴给用户）

```bash
adb -s <TV-IP>:5555 shell dumpsys package app.orchard.tv | findstr /C:"versionName" /C:"versionCode"
adb -s <TV-IP>:5555 shell settings get secure enabled_accessibility_services
adb -s <TV-IP>:5555 shell input keyevent KEYCODE_HOME
adb -s <TV-IP>:5555 shell dumpsys window | findstr mCurrentFocus
```

四条都要过：

1. `versionName` 是本次装的版本；
2. `enabled_accessibility_services` 里含橙龙桌面的守护服务；
3. 按 Home 之后；
4. `mCurrentFocus` 指向 `app.orchard.tv/...`（不是系统桌面、不是别的 launcher）。

不能用 ADB 时，让用户在电视上按 Home，看是不是回到橙龙桌面，并让他确认「控制中心 → 桌面守护」显示为已开启。

## 6. 常见故障

- **按 Home 还是回原桌面**：守护没开，或被系统省电策略关了（小米、雷鸟待机后会关无障碍）。重新打开守护；反复掉就说明这台机器只适合 S 版 + 替身。
- **开机回原桌面**：守护是进程级的，冷启动瞬间还没接上，属正常，几秒后会被拉回。用户在意就在设置里开「开机守护」。
- **装完图标不见了**：S 版声明了 HOME，装完可能被系统当成桌面；从设置 → 应用里找到 `app.orchard.tv` 打开一次。
- **电视上没有文件管理器**：通道 A/B 都不需要文件管理器；只有 U 盘路线才需要（用「小白文件管理器」）。

## 7. 回滚

```bash
adb -s <TV-IP>:5555 uninstall app.orchard.tv
```

卸载前提醒用户：桌面上的排列、壁纸、设置会一起消失。只想退守护的话，用 `--es reset true` 广播，或让用户在无障碍页里关掉「橙龙桌面守护」。
