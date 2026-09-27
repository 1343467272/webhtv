# AV-FREEZE-01 网盘播放周期卡死 / 黑屏 诊断记录

- 状态：本地修复、11 项测试及电视 arm64 安装包校验完成；真实 JAR 的 Android 运行与 TCL 长播复测待设备。以本文第 13 节为当前结论。
- 日期：2026-09-27（Asia/Shanghai, UTC+8）
- 分支 / HEAD：`main` @ `d187f6ae8b`
- 构建被测版本：`v5.6.0-202609241253`（versionCode 560）
- 现象（用户描述）：电视播放**网盘**视频时，每隔一段时间就卡死，甚至黑屏关机
- 关键修正 1：用户反馈**换 Exo 引擎同样过一段时间卡死** → 结论按**引擎无关**重新定位，不再归因 mpv
- 关键修正 2：用户报**真实黑屏并自动重启**并授权读取在线日志（`http://192.168.1.5:9978/debug/logs.txt`）→ 见第 6 节，**重启结论已被推翻并重新定性**
- 当前授权：用户已要求分析并修复 Bug；下方第 0–12 节保留早期诊断原文，其中已被新证据推翻的推断不再作为实现依据。

> **2026-09-27 更正**：日志 (38) 已记录主进程 ANR；电视重启后的进程观测到 739 个线程。实际加载的网盘 JAR 每次 `Pan.proxy()` 创建的下载线程池没有关闭，属于已确认的资源生命周期缺陷。旧文“未知协议禁止预加载就是根因”“没有 ANR/内存问题”“OTHER 一定是主进程退出”等结论撤回。详见第 13 节。

---

## 0. 结论速览

1. **根因是引擎无关的**。已直接观测到 **Exo 和 mpv 两个引擎在同一类网盘媒体上出现相同的"播放位置冻结 ~120 秒"**：
   - Exo：日志 (19) `21:16:02→21:18:19` 冻结 137s、`19:47:13→19:49:18` 125s、`21:22:02→21:24:06` 124s；日志 (17) 120s
   - mpv：日志 (26) `14:29:51→14:31:48` 冻结 117s

   这正是用户"用 exo 也一样卡死"的日志证据。

2. **冻结时长中位数 = 120 秒，且应用从不自愈**。全语料 14 个"观看中停滞"样本：最小 55s、中位 120s、最大 222s，其中 10/14 落在 110–140s。**每次停滞都以用户手动退出（`activity pause`/`stop`）结束，没有一次是应用自己恢复的。** 说明这个时长是**用户耐心**，不是任何超时机制。

3. **观看中并没有主线程死锁**（这是与早期判断的关键区别）。冻结期间主线程持续输出 `playback-progress`/`playback-disk-buffer`/`playback-telemetry`。所以"死机"是**媒体管线停摆 + 界面无反馈**，不是主线程被钉死。

4. **Exo 与 mpv 在网盘路径上共用同一套本地回环代理，并且都被判为 `protocol=unknown`**：
   - (19) Exo：`protocol=unknown` 88+54+48 次，`action=block reason=policy-block` 15 次，`reason=protocol-unknown` 9 次
   - (26) mpv：`protocol=unknown` 37+20 次，`action=block reason=policy-block` **130 次**，`reason=protocol-unknown` **51 次**

   即：路径无法识别 → 预加载被禁止 → 缓冲得不到补充 → 播放停摆。**引擎无关的共因。**

5. **"黑屏+自动重启"已定位为「应用进程被杀 → 0.6 秒后自动重启到主页」，不是电视断电重启**（2026-09-27 14:47 实测，见第 6 节）。决定性判据：设备开机时长 `elapsedRealtimeNs` **跨事故连续、从未归零**（旧进程末 37.7435 d → 新进程首 37.7439 d，+36.4 s），内核没有重启。真实发生的是：进程在被杀前 **35.5 秒内全线程零输出**（连 30 秒周期的后台定时器都停了），退出原因 `OTHER(13) importance=300 pss=41 MB`，`traceAvailable` 全部 `unavailable`（无崩溃栈）——即**被系统无痕杀掉**。

6. **"黑屏"的解释需要修正**：`keepScreen` 只在生命周期事件时被采样（`playback-lifecycle`，224 条），不是周期指标。观测到的规律是：`keepScreen=true` 仅出现在 `playing changed isPlaying=true` 之后，一旦暂停/缓冲即回到 `false`。**推断**：网盘停摆时播放器不再上报 `isPlaying=true`，常亮标志无法维持，电视空闲计时器随后息屏——用户看到"卡住然后黑屏"。

7. **mpv 另有一个独立的、真实的主线程 native 阻塞缺陷**（缺陷 A），它**不是**本次"网盘卡死"的共因，但会在 mpv 下叠加放大症状。

> 一句话：**卡死 = 网盘链路被判成"未知协议" → 预加载/补充被禁 → 位置冻结；而没有任何机制能发现"位置不再前进"，所以永远不会自愈。黑屏是停摆期间常亮标志无法维持。进程还会在被系统无痕杀掉前先全线程静默 ~35 秒，随后自动重启回主页。** mpv 还额外叠了一个主线程 native 死锁。

---

## 1. 证据分级说明

- **观测**：日志/代码里能直接读到的事实。
- **推断**：由观测推出的机制。

应用日志**无法**观测 SurfaceFlinger / AudioFlinger / HDMI / 内核 / lmk / 温控。因此"整机死机"只能部分证明。`READY`、首帧回调、音频写入被接受**都不等于**用户真的看到了画面或听到了声音。

---

## 2. 核心证据：引擎无关的"位置冻结"

### 2.1 方法

把日志切成"观看会话段"（`activity resume` → 下一次 `activity pause`/`stop`），**只在会话段内**统计 `positionMs` 保持不变的最长区间。这样可排除"用户暂停/离开"造成的假停滞。

### 2.2 结果（全语料 14 个样本，全部为网盘源）

| 日志 | 引擎 | 冻结区间 | 时长 | 位置 |
|---|---|---|---|---|
| (19) | **Exo** | 21:16:02 → 21:18:19 | **137s** | 140326 |
| (19) | **Exo** | 19:47:13 → 19:49:18 | 125s | 140326 |
| (19) | **Exo** | 21:22:02 → 21:24:06 | 124s | 42501 |
| (19) | **Exo** | 19:28:37 → 19:30:40 | 123s | 140326 |
| (19) | **Exo** | 19:41:26 → 19:43:27 | 121s | 140326 |
| (19) | **Exo** | 20:32:47 → 20:34:47 | 120s | 140326 |
| (19) | **Exo** | 21:18:37 → 21:20:35 | 118s | 292 |
| (19) | **Exo** | 19:26:32 → 19:28:27 | 115s | 140326 |
| (19) | **Exo** | 21:07:25 → 21:08:57 | 92s | 0 |
| (19) | **Exo** | 20:52:46 → 20:53:41 | 55s | 0 |
| (17) | **Exo** | 12:17:33 → 12:19:33 | 120s | 114747 |
| (9)(1) | Exo | 14:05:27 → 14:09:09 | 222s | 325050 |
| (9)(1) | Exo | 13:55:22 → 13:56:33 | 71s | 52988 |
| **(26)** | **mpv** | **14:29:51 → 14:31:48** | **117s** | **2037619** |

**两个不同引擎、同一种网盘源、同一种"约 2 分钟位置不动"的表现。** 这是缺陷与播放引擎无关的直接证据。

> (9)(1) 的两个样本引擎标记缺失（窗口内无 `kernel=` 行），但其会话为网盘源；即使剔除这两个样本，Exo 侧仍有 11 个样本，结论不变。

### 2.3 关键观测：每次都是用户手动退出

```
(19) 21:20:35.286 activity pause     ← 用户被卡 137s 后手动退出
(19) 21:24:10.890 activity pause     ← 124s 后手动退出
(19) 19:49:18.105 activity pause     ← 125s 后手动退出
(26) 14:31:51.923 activity pause     ← mpv 卡 117s 后手动退出
```

**观测：没有任何一次停滞由应用自身恢复。** 停滞时长统计（n=14）：最小 55s、中位 **120s**、最大 222s，10/14 落在 110–140s。时长分布集中在"人类耐心"区间，而非任何固定超时值 ⇒ **不存在发现并恢复停滞的机制**。

> 全部 14 个样本都发生在**会话段内**（已排除 pause/stop），且全部为网盘源；14/14 全部以用户手动 `activity pause`/`stop` 结束。

### 2.4 关键观测：冻结期间主线程是活的

以 (19) 的 137s 窗口为例，期间主线程持续输出（非静默）：

```
21:16:03.492 [main] playback-metrics: … pathTrust=blocked …
21:16:03.602 [ExoPlayer:Loader:ProgressiveMediaPeriod] okhttp-player: start … range=bytes=72937784060-
21:16:03.615 [OkHttp connect http://127.0.0.1:1314/...] connectStart host=loopback, proxyType=DIRECT
21:16:18.161 [main] playback-lifecycle: service startCommand …
```

全日志 `state` 分布：`BUFFERING 168`、`IDLE 106`、`READY 39`。

**因此：这不是主线程 ANR 型死机，而是"主线程活着、媒体位置不前进、界面没有任何恢复动作"。** 这正是 `PlaybackRecoveryMonitor`（`STALL_MS=8000`）**无法覆盖**的情形——它只监控**主线程心跳**，心跳正常，所以永不触发。

---

## 3. 共因：网盘路径被判为"未知协议"，预加载被禁

### 3.1 观测：两个引擎都命中同一分类

| 字段 | (19) Exo | (26) mpv |
|---|---|---|
| `protocol=unknown` | 88 / 54 / 48 | 37 / 20 |
| `action=block reason=policy-block` | 15 | **130** |
| `reason=protocol-unknown` | 9 | **51** |
| `playerPath=external-loopback` | 是 | 是 |

两个引擎都走 `playerPath=external-loopback`（本地回环代理：`http://loopback:local/…`、`127.0.0.1:1314`、`127.0.0.1:5266`、`127.0.0.1:9978`），且**都被判定 `protocol=unknown`**（网盘直链没有可识别的容器/协议特征）。

### 3.2 观测：链路本身在劣化

| 指标 | (19) Exo |
|---|---|
| `ProtocolException` | 40 |
| `unexpected EOF` | 34 |
| `pathTrust=blocked` | 58 |
| `pathTrust=limited` | 9 |
| `rebuffer` 时长 | 7561ms / 4700ms / 7022ms |
| 上游文件大小 | 最大 `contentLength=72693169982`（≈72.7 GB） |

样本：

```
playback-decision … domain=throughput outcome=observed old=64778960 … reason=path-blocked suppression=path-blocked
playback-metrics  … pathTrust=blocked preloadContended=true
playback-decision … domain=load-control … target=134217728 reason=guard
okhttp-player: failed … error=ProtocolException
exo-network: unexpected EOF recovered offset=… consecutiveAttempt=1
```

**推断的因果链**：网盘直链对超大文件（数十 GB）的随机 range 请求频繁半途断连 → 吞吐崩塌 → 路径信任被判 `blocked` → 负载控制收紧 → 由于同时被判定 `protocol=unknown`，**预加载被 `policy-block` 禁止**，缓冲无法补充 → 播放器停在 `BUFFERING`，位置不再前进 → 直到用户手动退出。

这也解释了为什么**换引擎无效**：问题不在解码/渲染层，而在两引擎共用的**取流与预加载策略层**。

---

## 4. 缺陷 A：mpv 主线程 native 阻塞（引擎专属，独立叠加）

### 观测（日志 21，2026-08-30，TCL MT9655，mpv）

看门狗输出**单调恶化、永不恢复**：

```
main stalled=1510ms   native=none                                      nativeElapsed=0ms
main stalled=4520ms   native=get-double:demuxer-cache-state/reader-pts  nativeElapsed=3751ms
main stalled=9548ms   native=get-double:demuxer-cache-state/reader-pts  nativeElapsed=8779ms
main stalled=14576ms  native=get-double:demuxer-cache-state/reader-pts  nativeElapsed=13807ms
main stalled=19619ms  native=get-double:demuxer-cache-state/reader-pts  nativeElapsed=18851ms
main stalled=24647ms  native=get-double:demuxer-cache-state/reader-pts  nativeElapsed=23878ms
main stalled=29682ms  native=get-double:demuxer-cache-state/reader-pts  nativeElapsed=28913ms
main stalled=34711ms  native=get-double:demuxer-cache-state/reader-pts  nativeElapsed=33942ms
main stalled=39740ms  native=get-double:demuxer-cache-state/reader-pts  nativeElapsed=38971ms
```

固定栈（行号来自**当时抓日志的构建**，原文照录）：

```
MPVLib.getPropertyDouble
 → MpvPlayer.mpvGetPropertyDouble(MpvPlayer.java:4686)
 → doublePropertyMs(4304)
 → refreshCacheState(3159)
 → refreshPlaybackState(3136)
 → Handler / Looper / ActivityThread.main
```

日志 (20) 是同类第二次，阻塞在 `native=get-string:current-tracks/sub2/id`，1506ms → 31986ms（`nativeElapsed=10061ms`）。

### 代码对应（当前 `main` @ `d187f6ae8b` 的行号）

- `MpvPlayer.java:4064` 在 `refreshCacheState()` (`:4046`) 内**同步**读取 `demuxer-cache-state/reader-pts`；
- 该读取经 `doublePropertyMs()` (`:5236`) → `mpvGetPropertyDouble()` (`:5722`) → `MPVLib.getPropertyDouble`，**全部在主线程同步完成**；
- 调用链 `refreshPlaybackState()` (`:4036`) ← `startStateRefresh()` (`:3879`)，**每 500 ms** 一次；
- 由提交 `46f5165391 Throttle MPV cache observer fallback` 引入。

> 注意栈里的行号（3159/3136/4304/4686）与当前源码（4046/4036/5236/5722）差异较大，说明抓日志的构建早于当前 `main`；定位时应以**符号名**为准。

**这确实是一个真实缺陷**，但它是 mpv 专属、且与第 2 节的"位置冻结"是不同签名（这里是主线程完全静默，第 2 节是主线程活跃但位置不动）。

### 为什么 Exo 卡死"看不见"

- 主线程看门狗**只存在于 MpvPlayer**（`startMainThreadWatchdog()` `:3912`、`runMainThreadWatchdog()` `:3947`，`onStall` 调用在 `:3964`，阈值 `MAIN_THREAD_STALL_THRESHOLD_MS = 1500` 在 `:101`）。
- 全仓库检索确认：**Exo 侧没有等价的主线程看门狗**（`ExoTunnelingProgressWatchdog` 仅覆盖 tunneling，`STALL_TIMEOUT_MS=3000`）。
- 且 mpv 停滞日志被 `SpiderDebug.isEnabled()` 门控（`:3965`）——**未开调试时连日志都不会留下**。

---

## 5. 已排除的解释

| 候选原因 | 排除依据 |
|---|---|
| 内存泄漏 / OOM 被杀 | 全语料检索无任何 `lowmemorykiller`/`lmkd`/`low memory`/`OutOfMemoryError`/`fatal`/`ANR`；`javaUsed` 12.3–214 MB 波动但不单调；`systemAvail` 稳定 808–1570 MB |
| 内存压力回收 | 所有 `playback-memory` 样本 `pressure=normal reason=system-normal systemLow=false` |
| 应用崩溃 | 退出记录为 `OTHER(13) status=0` / `EXIT_SELF` / `USER_REQUESTED`，无 fatal |
| 电视真的断电 | 开机时长跨 12 天单调递增（第 6 节） |
| 解码器不支持 | (19) `exo-capability: display=max1920x1080 … decoder=c2.mtk.hevc.decoder`，`exo-decoder-profile … failures=1` 仅 1 次 |
| 显示模式切换 | `domain=display-mode … reason=restore-not-needed / unknown-format`，未实际切模式 |
| 主线程 ANR | 冻结期间主线程持续输出（第 2.4 节） |

### 一处重要的自我纠错

早期把 (19) 中 `19:47:14 → 20:32:09` 的 2696 秒"位置冻结"当作卡死证据，**这是错的**：该窗口内有

```
19:49:18.105 activity pause   activity=VideoActivity …
20:32:08.263 activity resume  activity=VideoActivity …
```

即用户**主动暂停并离开**（看电视/走开），并非卡死。类似误判还有：

- 21:03:57→21:05:27 的 90 秒空档：对应 `csp-warmup: reset generation=2/3` + `jar-loader: clear loaders=1 spiders=101`（UI 导航）
- `webhtv-debug-log (9) (1).txt` 的 397 秒空档：以 `13:56:44 activity pause` 开始、`14:03:21 调试日志已恢复` 结束

**教训：长静默/长冻结必须先排除 pause/stop/导航，否则会把用户行为误判为缺陷。** 本文第 2 节的结论已按"仅会话段内"重新统计。

---

## 6. 2026-09-27 14:47 事故：黑屏与"自动重启"的真实性质

本节是对早期结论的**推翻与重新定性**。早期版本写的是"电视从未真正重启，所以关机是待机/息屏"——**用户明确指出确实发生了黑屏并自动重启**。原文拿 12 天跨度的开机时长单调性去否定一次几十秒的事件，方法上不成立。本轮改为**直接跨事故比对开机时长**，结论随之改写。

被测设备（全部同一台、同一构建）：

```
manufacturer=TCL brand=TCL model=Smart TV Pro device=MT9655
product=MT9655_4K_CN android=14 sdk=34 incremental=AU01 supportedAbis=arm64-v8a
app=5.6.0(560)
```

### 6.1 决定性判据：内核没有重启，是**应用进程**死了

`elapsedRealtimeNs` 是设备开机时长（单调时钟，**只在真实重启/断电后归零**）。跨事故采样：

| 采样点 | 开机时长 |
|---|---|
| 旧进程（`88221d5c…`，pid 29472）最后一条 | **3261038.9 s = 37.7435 d** |
| 新进程（`a358ad41…`，pid 10348）第一条 | **3261075.3 s = 37.7439 d** |
| 差值 | **+36.4 s**（连续，从未归零） |

`captureStartedAtNs` 区间 3260993.5 s → 3261514.7 s（37.7490 d）同样连续。

> **判定**：电视**内核/系统没有重启**。用户看到的"自动重启"是**应用进程被杀后在 0.6 秒内被系统自动重新拉起**，界面回到主页。两者观感几乎一样，但性质完全不同——这是应用级故障，不是整机故障。

自动拉起能成立的原因：`ManageService` 返回 `START_STICKY`（`service/ManageService.java:202`），Android 会在进程被系统回收后重建它，进而拉起 `App`。

### 6.2 事故时间线（同一份日志内跨两个进程）

| 时刻 | 进程 | 事件 |
|---|---|---|
| 14:46:25.160 | 旧 | 本轮日志窗口开始（`restored-history` 段） |
| 14:46:51.847 / .858 / 14:46:58.992 / .998 | 旧 | `playback-memory` 后台定时器（30 秒周期）最后 4 条输出 |
| 14:47:04.739 | 旧 | `proxy-stream: open id=515 do=pan range=bytes=1819-` ——**至今未 close** |
| 14:47:10.920 | 旧 | **旧进程最后一条日志**：`exo-preload: event=helper-prepared session=6 generation=23 prepareMs=5475 cacheBytesAdded=0` |
| 14:47:10.905–14:47:10.920 | 旧 | 全线程在 **1.15 s 内同时停止**（`main` / `PreCacheHelper:Loader` / `NanoHttpd #1148` / `OkHttp` / `ExoPlayer:Playback`） |
| 14:47:46.407 | — | 死亡墙钟（由新进程回读 `ApplicationExitInfo` 得到） |
| 14:47:47.002 | 新 | **新进程第一条日志**（重启间隔 **0.6 s**） |
| 14:47:47.029 | 新 | `env: reason=restore` |
| ~14:47:48 | 新 | `startup: home first frame cost=1120ms` → **落地在主页，播放未恢复** |

事故前最后状态：引擎 **Exo**，trace `p-1hxj4r0-6`，session 6，媒体 `玩偶@@@/voddetail/80275.html@@@1`（**本地源**），`state=3`、`isPlaying=true`、`keepScreen=true`、position ≈ 5035 ms。

### 6.3 死亡签名：全进程静默 35.5 秒，然后被无痕杀掉

| 观测项 | 值 | 含义 |
|---|---|---|
| 最后日志 → 死亡墙钟 | 14:47:10.920 → 14:47:46.407 = **35.5 s** | — |
| 这 35.5 s 内的事件数 | **0**（按 `wallTime` 排序核验，非日志管道假象） | 不是"只主线程卡"，是**全进程停摆** |
| 30 秒周期后台定时器 | 14:46:58.998 之后再无输出（本应约 1.6 次） | 连后台定时器都停 → 不是主线程阻塞签名 |
| 全线程最后输出跨度 | **1.15 s** 内齐停 | 与"SIGSTOP/cgroup freezer 式冻结"一致 |
| 在途 HTTP 流 | `id=515` 打开 6.18 s **从未 close**（本段共开 11 个流，仅此 1 个悬空） | 网络传输被**中途切断**，不是自然收尾 |
| 退出原因 | `OTHER(13) status=0 importance=300 timestamp=1790491666407 pss=41370 rss=82140` | 系统回收，非崩溃、非 ANR、非 OOM、非用户操作 |
| `traceAvailable` | `{"status":"unavailable"}` | 无崩溃栈 → **无痕被杀** |
| `recovered-journal` | player-2 / controller-1 / player-1 **全部 `unfinished=true`** | 突然死亡，无优雅退出 |

`importance=300` = `IMPORTANCE_SERVICE`，结合 41 MB 的低 pss，与"系统在资源/后台策略下回收该进程"一致。

> **重要限定**：`OTHER(13)` 是 `ApplicationExitInfo` 的归类，**不携带具体原因**。日志中没有 `SIGNALED`/`lmkd`/`lowmemory`/`ANR in`/`native crash`/`fatal`/`crash` 任何一条，且整份日志 876 条事件**全部 `level=info`**。因此"**谁**杀了它"应用日志无法回答；能确证的是"**它被杀死了，且死前已全进程静默 35.5 秒**"。

### 6.4 这是复现性签名，不是孤例

全语料中含死亡记录的 13 份日志里，**10 份是同一个 `reason=OTHER(13) importance=300` + 低 pss（40–55 MB）签名**：

| 日志 | 退出原因 | importance | pss | 死前静默 | 重启间隔 |
|---|---|---|---|---|---|
| (19) | `OTHER(13)` | 300 | 54.7 MB | 1.6 s | 15.9 s |
| (24) | `OTHER(13)` | 300 | 40.8 MB | 0.3 s | 148.4 s |
| (25) | `OTHER(13)` | 300 | 40.8 MB | 0.3 s | 148.4 s |
| (26) | `OTHER(13)` | 300 | 40.9 MB | 1.6 s | 81.4 s |
| (34) | `OTHER(13)` | 300 | 54.9 MB | 2.5 s | 2.7 s |
| (37) / htv2 / htv3 | `OTHER(13)` | 300 | 40.4–41.4 MB | **35.5 s** | **0.6 s** |
| (9)(1) | `OTHER(13)` | — | 0 | 0.1 s | 396.8 s |
| (36) | `USER_REQUESTED(10)` | 125 | 709.9 MB | — | — |
| (27) | **`SIGNALED(2) status=9`** | 100 | 348.4 MB | — | 0.4 s |
| (20) | **`EXIT_SELF(1)`** | 400 | 58.4 MB | 5.2 s | 2.8 s |

**没有任何 OOM 记录** ⇒ 卡死不是被系统低内存杀掉。`USER_REQUESTED(10)`(36) 与 `SIGNALED(2)`(27) 是另外两种性质（用户强停 / 外部 SIGKILL），应与 `OTHER(13)` 系列区分。

### 6.5 恢复机制全程没有生效

`PlaybackRecoveryActivity` 属于 `:playback_recovery` 进程，本意就是在播放异常时接管并引导用户。本次事故中：`playback_recovery` 出现次数 = **0**，`PlaybackRecoveryMonitor` 也没有留下任何记录。

原因是结构性的：`PlaybackRecoveryPolicy.STALL_MS = 8000`（`player/mpv/PlaybackRecoveryPolicy.java:5`）监控的是**主线程心跳**（`PlaybackRecoveryMonitor.java:61/66`）。本次故障有两种形态，它**一种都覆盖不到**：

1. 观看中的**位置冻结**——主线程仍在正常输出（第 2 节），心跳健康，看门狗看不见；
2. 死亡前的**全进程 35.5 秒静默**——进程即将被杀，没有机会执行任何恢复逻辑。

而且 `PlaybackRecoveryActivity` 只在**用户主动点击**时才 `Process.killProcess(...)`（`ui/activity/PlaybackRecoveryActivity.java:85-105`），从不自动介入。

回到主页而不是回到播放页，是因为重启走的是 `App.onCreate` → `startBackgroundServices`（`App.java:99-110`）的常规冷启动路径：`env: reason=restore` 只恢复**日志**，不恢复播放会话。`recovered-journal` 里的 player-2 停在 `stage=media3-position-advancing`、`positionMs=137`——数据在，但没有代码去用它续播。

### 6.6 黑屏机制（部分推断）

`PlaybackActivity.java:738-739`：

```java
if (isPlaying) addFlags(FLAG_KEEP_SCREEN_ON);
else if (!isBuffering()) clearFlags(FLAG_KEEP_SCREEN_ON);
```

`isBuffering()` = `getPlaybackState() == Player.STATE_BUFFERING`（`:175`）。

观测到的 `keepScreen` 规律：`true` 只在 `playing changed isPlaying=true` 之后出现；暂停、缓冲、离开后回到 `false`。(19) 中 `keepScreen=false` 155 次、`true` 69 次。注意 `else if (!isBuffering())` 的写法意味着 **`BUFFERING` 状态下不会清除该标志**。

**推断**：网盘停摆时 `isPlaying` 不再为真且状态非 `BUFFERING`，常亮标志被清除，电视空闲计时器到期后息屏；紧接着进程被杀 → 用户感受为"先卡住、再黑屏、然后重启"。注意 `keepScreen` 只在生命周期事件时采样，是**状态快照**，不能当作连续监控指标。

> **诚实边界**：应用日志**没有** SurfaceFlinger / AudioFlinger / HDMI / 内核 / 温控 / lmk 任何一行，因此**无法从本证据排除**面板级电源循环或 HDMI 瞬断。本节结论严格限定为：**应用侧观测到的是一次进程死亡 + 自动重启，而非内核重启**。

---

## 7. 观测：网盘链路的吞吐量两极分化

本地回环代理（NanoHTTPD 2.3.1，`DefaultAsyncRunner`，见 `server/`）的 `proxy-stream` 记录了每条流的实际速度。在 14:31:44 → 14:33:44 的 `do=pan` 阶段（**412 条流 / 1293.9 MiB / 平均 10.78 MiB/s**）内部，存在**两类截然不同的流**：

| 类型 | 特征 | 条数 | 中位速度 | 总字节 |
|---|---|---|---|---|
| **探测流** | 开区间 `range=bytes=N-`（探到文件末尾） | 214 | **0.050 MiB/s** | 121.5 MiB |
| **分块流** | 闭区间 `range=bytes=A-B`（顺序拉取） | 198 | **2.610 MiB/s** | 1172.4 MiB |

**中位速度相差约 52 倍。** 214 条探测流中有 **64 条低于 0.05 MiB/s**；最小响应仅 **18 字节**、耗时 3–7 ms（在 `range=bytes=1092009850-` 上，即**探到 1 GB 偏移处**）。

请求呈现交替模式：`range=bytes=1067636397-`（探测到 EOF）与小块顺序请求（`bytes=648843860-661943875`、`bytes=661943876-668411377` …）**交错出现**。

> **推断**：播放器/代理在反复对"远超出实际需要"的偏移做开区间探测（最远探到 1 GB 之后），这些探测绝大多数瞬间失败或极慢，且与真正拉数据的分块流**争抢同一条网盘链路**。这既能解释"链路本身在劣化"（第 3.2 节），也解释了为什么缓冲迟迟补不上——**带宽被探测消耗掉了**。

### 7.1 本地源也会致命失败

同一份日志里，一次**本地源**播放（trace `p-1hvl0yu-1`，key `玩偶@@@/voddetail/80275.html@@@1`）在播放到 83% 时出现确证的致命错误：

```
14:03:04.287 playback-metrics: trace=p-1hvl0yu-1 error code=ERROR_CODE_IO_READ_POSITION_OUT_OF_RANGE errorType=b21
14:03:04.309 playback-error: stage=network-io confidence=confirmed evidence=media3-error-code
                errorCode=ERROR_CODE_IO_READ_POSITION_OUT_OF_RANGE
                route=APP_LOCAL_SERVICE owner=app-main-server loopback=true
                observedLeg=app-to-owned-local-service controlScope=app-owned-service-code action=FATAL
```

会话 `duration=1790755 ms`，错误发生在 `position=1496813`（约 83%）。

> 这条**不是**网盘问题（`route=APP_LOCAL_SERVICE`、`loopback=true`、`controlScope=app-owned-service-code`），但同一份 trace key 也出现在 14:47 事故的旧进程里。它证明本地回环代理这一层本身存在**位置越界读取**的真实缺陷，值得与第 3 节分开跟踪。

---

## 8. 共用主线程阻塞隐患（次要，会放大症状）

这些每 5 秒的周期任务跑在主线程，实测单次停顿 1.5–2.1 秒：

| 位置 | 行为 |
|---|---|
| `MpvHlsCacheCoordinator.physicalBytesLocked:310` | `directory.listFiles()` + `file.length()` 逐文件 stat |
| `PlayerManager.evaluateMpvCaches:3663` | 由 5 秒 telemetry 触发 |
| `PlayerManager.publishPlaybackTelemetry:6421` | `PLAYBACK_TELEMETRY_INTERVAL_MS = 5000`（`:150`） |
| `Util.getSerial:144` | `Shell.exec("getprop ro.serialno")` fork+exec（有 static 缓存） |
| `PlaybackRecord.clientKey():384` → `Device.get().getUuid()` | 会话记录路径 |

实测样本：(30) `MemoryIntArray.nativeGet`/`Settings$GenerationTracker.readCurrentGeneration` **2148 ms**；(32) `AudioTrack.native_start … MPVLib.setPropertyBoolean` **1612 ms**、`SQLiteConnection.nativeExecuteForCursorWindow` **1728 ms**；(33) `MpvHlsCacheCoordi…` **1710 ms**；(34) **1546 ms**。

**判定：加剧因素，不是主因**——它们无法解释第 2 节"主线程活跃但位置冻结"的签名。

---

## 9. 无法从应用日志证明的部分

1. 无法观测 SurfaceFlinger / AudioFlinger / HDMI / 内核 / 温控 / lmk，"整机死机"只能间接推断。
2. 无法证明用户"看到/听到"了内容。
3. 第 3.2 节因果方向（链路劣化 → 停摆，还是停摆 → 链路超时）需一次带诊断的复现来定序。
4. mpv 的 native 阻塞（第 4 节）是否与网盘特定 URL 相关，日志不足以判定。
5. **无法回答"是谁杀了进程"**：第 6.3 节只能证明进程被系统无痕回收（`OTHER(13)`），`traceAvailable` 全部 `unavailable`；日志中没有 lmk / 温控 / 后台策略任何一行。若要定位到具体触发者，需要 `dumpsys activity exit-info` 或 logcat 系统侧日志。
6. **无法排除面板级黑屏**：应用日志不含显示子系统信息，第 6.6 节的"黑屏 = 息屏 + 进程被杀"是推断而非观测。

---

## 10. 建议的下一步（仅取证，不改代码）

**最小且决定性的一步**：**开启诊断**后复现一次，在卡住当下立刻打点导出。

当前状态复现等于没有证据：日志 (36) 完全没有可关联播放事件——`"truncated":1`、`"diskBytes":0`、`"scope":"memory-window"`、`"exportFallback":"writer-timeout-or-error"`，报告原文即"没有可关联的播放事件"；而 mpv 停滞日志还被 `SpiderDebug.isEnabled()` 门控。

复现时需确认：

1. 卡住期间主线程是否仍在输出？（区分第 2 节"位置冻结"与第 6.3 节"全进程静默"两种签名）
2. `protocol`/`pathTrust`/`reason=protocol-unknown`/`policy-block` 是否同时出现？
3. `keepScreen` 何时由 `true` 变 `false`？息屏是否紧随其后？
4. 是否出现 `ProtocolException` / `unexpected EOF` 与 `path-blocked`？
5. **`proxy-stream` 的探测流（开区间 `range=bytes=N-`）是否在事故前激增？** 第 7 节显示它们的中位速度只有分块流的 1/52。

**若要定位杀进程的主体**（应用日志给不出答案），需要在电视上另取系统侧证据：

```bash
adb shell dumpsys activity exit-info com.fongmi.android.tv
adb logcat -b all -d | grep -Ei 'lmkd|lowmemory|low memory|kill|ActivityManager'
```

---

## 11. 后续修复方向（需用户明确批准后才动代码）

按"最小且可独立回滚"排序，**本轮不实施**：

1. **引擎无关的"播放位置停滞"看门狗**（针对根因）：不依赖主线程心跳，检测"本应播放但位置长时间不前进"。这是唯一能同时覆盖 Exo 与 mpv 的缺口，也是第 6.5 节 `STALL_MS` 失效的直接补丁。
2. **放宽 `protocol=unknown` 路径上的预加载限制**：网盘直链天然无法识别协议，当前被判 `policy-block` 直接禁止补充缓冲，等于放弃了唯一能让播放继续的机制。需评估风控/流量与限流风险。
3. **修掉探测流的带宽浪费**（第 7 节）：避免对远超播放位置的偏移（观测到探到 1 GB 之后）反复做开区间探测，把带宽让给顺序分块。
4. **mpv 同步 native 属性读取移出主线程 / 加超时**（缺陷 A）：`MpvPlayer.java:4064` 及同类调用。
5. **Exo 侧补主线程看门狗**：让 Exo 卡顿也能留下证据（对齐 mpv 的 `mpv-anr`）。
6. **停摆期间维持常亮**：`PlaybackActivity.java:738-739` 增加宽限期，避免缓冲时息屏导致用户误判为"死机/关机"。
7. **重启后续播**：新进程落在主页且播放不恢复（第 6.2 节）。`recovered-journal` 已有 `positionMs`/`stage` 数据，但没有任何代码消费它去续播。这是把"重启"从用户可见故障降级为透明恢复的关键一步。
8. **查清 `ERROR_CODE_IO_READ_POSITION_OUT_OF_RANGE`**（第 7.1 节）：发生在 `route=APP_LOCAL_SERVICE` 上，属自有回环代理的越界读取，与网盘问题独立。

---

## 12. 复现与取证清单

- [ ] 设置中**开启诊断/调试日志**（否则 mpv 停滞无记录）
- [ ] 分别用 Exo 与 mpv 各复现一次
- [ ] 卡死当下立即在应用内打点（触发 incident 窗口）
- [ ] 导出后确认 `completeness` 不是 `partial`、`exportFallback` 为空
- [ ] 记录电视型号 + 开机时长，按第 6.1 节方法**跨事故比对**（而非跨多日比对）
- [ ] 记录卡住→手动退出的实际等待秒数，与日志冻结时长核对
- [ ] 事故后另取 `dumpsys activity exit-info` 与系统侧 logcat（第 10 节）

---

## 附：证据文件

| 文件 | 用途 |
|---|---|
| `webhtv-debug-log (19).txt` | **Exo + 网盘主证据**：4 次 ~120s 位置冻结、`protocol=unknown`、链路劣化、`keepScreen` |
| `webhtv-debug-log (26).txt` | **mpv + 网盘主证据**：117s 位置冻结、`policy-block` 130 次（引擎无关的决定性对照） |
| `webhtv-debug-log (17).txt` | Exo + 网盘，120s 冻结（独立复现） |
| `webhtv-debug-log (21).txt` | mpv 主线程 native 阻塞（`reader-pts`，至 39.7s，缺陷 A） |
| `webhtv-debug-log (20).txt` | mpv 第二次 native 阻塞（`current-tracks/sub2/id`，至 32.0s） |
| `webhtv-debug-log (27).txt` | 唯一被外部 SIGKILL（`status=9`） |
| `webhtv-debug-log (29)(30)(32)(33)(34).txt` | 开机时长链（第 6.1 节的连续性对照） |
| `webhtv-debug-log (36).txt` | 无播放事件、导出降级 |
| `webhtv-debug-log (35).txt` | **另一台电视**（Android 11 / v7a），不可混用 |
| `webhtv-debug-log (37).txt` | **14:47 事故**（14:47:53 抓取）：进程死亡 + 重启，旧进程回读 `OTHER(13)` |
| `htv1`（在线 `14:33` 抓取） | 网盘吞吐量化（第 7 节）：探测流 vs 分块流 52× 差距；`READ_POSITION_OUT_OF_RANGE` |
| `htv3`（在线 `14:55` 抓取） | **决定性**：跨事故开机时长连续（第 6.1 节）、35.5s 全进程静默、悬空流 `id=515` |

### 自我纠错记录

| # | 早期判断 | 为什么错 | 修正 |
|---|---|---|---|
| 1 | 归因 mpv 主线程死锁 | 用户指出换 Exo 同样卡死 | 改为引擎无关的"位置冻结"（第 2 节） |
| 2 | (19) 2754s 位置冻结当成 hang | 区间内含 `19:49:18 activity pause` → `20:32:08 activity resume`，是用户暂停离开 | 只统计会话段内（第 2.1 节） |
| 3 | 用文件顺序当时间顺序，算出"主线程静默 1107.8s" | 日志是轮转分段拼接，**非时间有序** | 先按时间戳排序再算间隔 |
| 4 | "电视从未真正重启，是待机/息屏" | 拿 12 天跨度的开机时长单调性去否定一次几十秒的事件，方法不成立；用户明确说确实黑屏重启了 | 改为**跨事故**比对开机时长，得出"内核未重启、应用进程被杀 + 0.6s 自动重启"（第 6 节） |

## 13. 2026-09-27 新事故、结论更正与修复

### 已建立的证据

- 用户确认 TCL MT9655，Exo / MPV 均受影响，低分辨率视频也见 CPU 约 150%；电视无 ADB。该 CPU 数字是用户观测，尚无线程 CPU 采样，不能据此归因解码器。
- `(38)` 及在线日志：19:16:46.584，前进程退出 `ANR(6)`，VideoActivity 按键分发等待 12074 ms，PSS 1065397 KiB；ANR 前缀显示 VmSwap 658144 KiB。前进程 PID 10348，新进程 PID 30688。跨事故开机时间连续，仅能确认本次应用进程重启，不能据此否定用户在其他时刻看到整机异常。
- ANR 输出仅保留 8192 字节，debuggerd 抓栈超时，没有可用的主线程栈。`ptrace_stop` 是抓栈阶段的状态，不能当作阻塞根因。
- 约 19:23 从电视 `/file/proc/self/task` 的目录列表观测到 **739 个线程**。下载的实际 JAR 为 2044180 字节，SHA-256 `143c93e91cd88bebbad57e8a1ffe23f4acb1904bc9ca3189819e522d1b33f655`，缓存键 `9754bc181ef423f46b8c3d7f1a565c06`。
- 19:45 同一目录的后续只读采样仍有 724 个线程；两个快照不能单独证明增长速率，但说明高线程数量持续存在。
- 对该 JAR 的 DEX 反汇编确认调用链：`spider.Proxy.proxy → Pan.proxy → merge.m.f.<init>/e`。每个代理请求新建固定下载线程池，默认 16 线程；每个流最多提前缓存约 70 MiB。`Init.execute` 使用另一个共享的 5 线程池输送数据到 PipedInputStream。
- `merge.m.f.b` 的结束路径只置停止标志并关闭输出管道，**全部路径都没有 shutdown 下载线程池**。下游关闭输入流也没有直接通知下载对象；若 writer 在等分块或尚未排到执行，下载资源继续存活。读取完成、Range 探测、重连、预加载都会放大该缺陷。这证明资源泄漏，不单独证明每一次黑屏/CPU 高占用都由它引起。
- 事故前 Exo 预加载发生多次探测、取消和重试，最后出现空代理响应 HTTP 500；它会增加有缺陷代理的调用次数，但不是已经证实的独立根因。

### 对旧分析的更正

1. `protocol=unknown / policy-block` 限制的是可选预加载，不会禁止正常前台读取；不能推出“缓冲不再补充”。
2. `(26)` 所谓 117 秒冻结同时有 `isPlaying=false`、`preload-pause paused=true`，缓冲增长至 300 秒；`activity resume` 区间不能排除播放器暂停。`(19)` 部分窗口为失败后的 IDLE，不能统计成持续播放卡顿。
3. 旧 `OTHER(13)` 含 `[ISOLATED NOT NEEDED]`，可能是 WebView 子进程。现有退出日志没有关联进程名，不能把它作为主进程退出原因。
4. `(38)` 已直接反证“没有 ANR”和“没有明显内存问题”。不实施旧文基于这些假设提出的强制预加载、自动重启或常亮改动。

### 最小修复设计与验收

- 本轮是已有 JAR 加载兼容边界的资源生命周期修复，不更新 Exo/MPV/native 依赖，也不重写网盘协议。
- 比较方案：不改会持续泄漏；离线修改远端 JAR 会被源更新覆盖且改变二进制维护归属；选用 App 现有 `JarLoader` 兼容钩子，**仅精确 SHA-256 命中这版 JAR 时启用**，未知版本原样加载。
- 在 `Init` 的共享 executor 外加委托，识别经验证的管道 writer 及其下载对象；仍由原 executor 执行原任务。代理调用的请求局部上下文将该对象与返回的输入流关联；EOF、读异常、close、代理调用失败、writer 结束和 loader clear 都补做幂等释放（停止标志、shutdownNow、关闭管道、清理待输出分块）。其他 JAR 任务与解析、Range、鉴权、线程数和内容数据路径保持原语义。
- 不新增轮询线程，不在主线程等待线程池退出；在途网络读取可能要等待原客户端超时，测试分别验证正常完成、取消及失败后的最终退出。
- 验收：重复请求/关闭后下载线程池全部终止；取消尚未执行的 writer 也能释放；失败和并发请求互不串流；原响应状态、头和字节保持；未知 JAR 不启用；电视 arm64 APK 构建通过。电视长时间播放、CPU 与 ANR 的回归仍需安装后实测。
- 允许路径：`JarLoader.java`、`PanProxyCompat.java`、对应 JVM/Android 测试、本文。原始未跟踪文档已备份至 `/private/tmp/webhtv-av-freeze-20260927/AV-FREEZE-01-original.md`，采用 `--adopt-dirty` 保留并续写；其余初始脏路径全部受保护。
- 回退基线：`main` / `d187f6ae8bfaf0a4720724281d0a91186c57f73d`。修复通过 guard `av-freeze-01-pan-proxy-lifecycle` 原子提交并生成 annotated `recovery/av-freeze-01-pan-proxy-lifecycle/*` 标签；撤回该任务提交或安装原版 APK 即可回退，未修改电视配置或远端 JAR。

### 实现与验证记录

- `JarLoader.ProxyMethod` 同时持有反射入口与兼容对象，避免 clear 与在途请求竞争时丢失回收上下文；只对已匹配 JAR 的 `do=pan` 调用建立资源作用域。
- `PanProxyCompat` 已实现版本匹配、原 executor 委托、请求绑定、writer/stream/loader 的幂等释放。共享 writer 线程的取消只针对仍在执行该任务的线程，并在复用前清除本次取消标志。
- 19:39：`PanProxyCompatTest` **11 项通过，0.469 秒**。验证字节与响应头透传、正常 EOF、writer 自然结束、首块前取消、排队中取消、代理异常、无效响应、executor 拒绝、读/close 异常、12 路并发隔离与重复 close、loader clear、无关任务与未知同尺寸 JAR。命令使用 JDK/JUnit 直接编译运行这两个纯 Java 文件，日志 `/private/tmp/webhtv-av-freeze-20260927/jvm-tests.log`。
- 19:42：真实 JAR 的 `PanProxyCompatDeviceTest` 已编译并经 D8 生成 ART 测试 DEX（min API 26）。程序先以本地 HTTP Range 数据复现原 JAR 的 4 个残留线程，再运行 16 次完整/取消请求验证回收与字节一致性；**尚未执行，不能把预期结果当成通过**。
- Android 实测环境限制：当前 `adb devices` 无设备；现有 API 36 电视模拟器的 `system.img` 缺失，启动失败。已向用户询问是否可接回此前的 vivo 手机，不修改电视或重新登录网盘。
- 19:49：`:app:assembleLeanbackArm64_v8aDebug` **BUILD SUCCESSFUL，6m47s**，包含最新 `JarLoader.ProxyMethod`。为保护初始 `app/.cxx/`，CMake staging 定向到 `/private/tmp/webhtv-av-freeze-20260927/cxx`。构建日志 `/private/tmp/webhtv-av-freeze-20260927/apk-build.log`。
- 测试包：`/private/tmp/webhtv-av-freeze-20260927/webhtv-av-freeze-01-leanback-arm64-v8a.apk`，180695630 字节；SHA-256 `ec4d56b8b07396436a1402971d276db48683d1d9c081dfd031e0669ce97c44eb`。ZIP 完整性、APK 签名、仅 arm64-v8a native 库及兼容类打包均通过。构建时 Git 身份为回退基线加本次工作区改动，产物以上述摘要关联。
- 电视复测时先确认日志 `pan proxy lifecycle compat enabled`，再观察反复拖动/换集、退出及长播后线程是否回落，分别检查 Exo/MPV。CPU 150% 和黑屏/ANR 是否全部消失目前没有实测结论；遇到其他 SHA 的 JAR 时不会启用此补丁。

### Recovery anchor

- 目标：已实现对已确认网盘 JAR 下载线程池泄漏的兼容修复，验证生命周期并产出电视 arm64 测试包；不把未验证的电视症状宣称为已治愈。
- guard：`av-freeze-01-pan-proxy-lifecycle` / quick-fix，已启动；初始 `.codex-resume/`、`app/.cxx/`、`codex-resume` 受保护。
- 已完成：新事故取证、实际 JAR 调用链与缺失 shutdown 确认、原文备份、兼容层实现、11 项 JVM 测试、App/ART DEX 编译、电视 arm64 APK 构建及完整性/签名校验。
- 本次原子变更：`JarLoader.java`、`PanProxyCompat.java`、JVM/Android 测试与本文；guard 范围不变，提交及恢复标签以该任务的 guard 收口记录为准。
- 剩余风险：Android 实际 JAR 运行测试尚缺设备；电视无 ADB，长播/CPU/ANR 的最终回归未完成。模拟器缺镜像属于环境失败，非代码回归。
- 唯一下一步：设备可用后运行已准备的真实 JAR ART 生命周期测试。
