# 蓝易聊 · Bluetooth Easy Talk · `io.txtbt.chat`

一个**扁平、无图标、纯文字排版**的安卓聊天应用。它不联网、不注册、不用服务器：
**每一条消息都是一次真实的 `.txt` 文件传输**，走蓝牙 RFCOMM（SPP）串口通道。

- 应用名：中文系统下显示 **蓝易聊**，其他语言显示 **Bluetooth Easy Talk**
- 当前版本 **0.0.11**，安装包 **72.7 KB**，`minSdk 14` → **Android 4.0 及以上**均可运行
- **零第三方依赖**：没有 AndroidX、没有 AppCompat、没有 Material、没有混淆器
- 设置页可切换 **暗色 / 浅色 / 跟随系统** 主题，并可设置**自定义聊天背景**
- 支持 **中文 / English / 日本語 / Русский** 四国语言（应用内切换，各 92 条完整翻译）
- 界面里没有任何图标或图片资源：全部由文字、色块和 1dp 细线构成

---

## 0. 本次更新（0.0.11）

| 改动 | 说明 |
|---|---|
| **取消收到文件时的弹出提示** | 消息到达不再弹 Toast；列表自己刷新、状态行反映连接，只有用户主动操作失败时才提示 |
| **后台收到消息 → 常规系统通知** | 应用不在前台时，用一条普通可清除通知显示最新消息（标题=发送者，正文=内容预览，点击进入聊天） |
| **自定义聊天背景**（可选任务） | 设置页可选一张图片作为消息区背景，带主题色蒙版保证两种主题下文字都可读；可随时移除 |
| **更名** | `蓝易聊`（中文）/ `Bluetooth Easy Talk`（英文、日文、俄文） |
| **版本号** | `1.0.0 → 0.0.11`（versionCode `1 → 11`） |
| **不覆盖旧版本** | 每个版本输出到独立目录，旧 APK 原样保留 |

上一版 `dist/txtchat-1.0.0.apk` 仍然保留在工作区里，没有被覆盖。

---

## 1. 产物位置

```
dist/txtchat-1.0.0.apk                                  旧版本，原样保留
releases/v0.0.11/bluetooth-easy-talk-0.0.11.apk         当前版本（0.0.11）
```

`build.ps1` 现在按版本号建目录，以后每次改 `$versionName` 都会进新目录，**不会覆盖任何历史版本**。

```powershell
adb install -r releases\v0.0.11\bluetooth-easy-talk-0.0.11.apk
```

---

## 2. 核心理念：聊天记录就是两个文件夹

```
<应用目录>/txtchat/
├── outbox/   我发出去的每一条消息，各占一个 .txt 文件
├── inbox/    收到的每一条消息，各占一个 .txt 文件
└── *.txt     导出的整段对话记录（可直接分享）
```

**文件名 = 元数据，文件内容 = 消息正文。**

```
outbox/20240601-120430_我的手机.txt     内容：晚上七点在地铁站见
inbox/20240601-120502_小明.txt          内容：好的，我提前十分钟到
```

所以：

- 任何一个 `.txt` 都能用记事本/任何文本编辑器打开，**不需要本应用**
- 删掉一个文件就等于删掉一条消息，没有隐藏数据库
- 收到的文件与发出去的文件**逐字节一致**

---

## 3. 蓝牙传输协议

一次连接 = 一条 RFCOMM 通道（标准 SPP UUID `00001101-...-00805F9B34FB`）。
连接建立后每发一条消息就传一个「文件帧」：

```
HELLO/1 <我的显示名>\n                       ← 连接建立时各发一次
TXTCHAT/1 <字节数> <文件名>\n                ← 帧头
<字节数 个原始字节，即 .txt 文件的完整内容>    ← 帧体
```

要点：

- **长度前缀**分帧，所以正文里出现换行、甚至出现 `TXTCHAT/1` 字样都不会破坏协议
- 正文以 UTF-8 原样落盘，收件箱里的文件与发件箱里的文件完全相同
- 单条消息上限 256 KB；畸形帧头、截断帧体会被明确拒绝而不是静默出错

这套编解码逻辑全部在 `TxtCodec.java` 里，**不依赖任何 `android.*`**，
所以它既能编进 APK，也能在电脑上直接跑单元测试（见第 8 节）。

---

## 4. 消息提示行为（0.0.11 的重点改动）

| 场景 | 行为 |
|---|---|
| 应用在前台，收到消息 | **不弹任何东西**。消息直接出现在列表里并自动滚到底部，顶部状态行显示频道状态 |
| 应用在后台，收到消息 | 更新一条**常规系统通知**：标题是发送者名，正文是消息预览，点击回到聊天，可滑动清除（连续多条会覆盖同一条通知，始终显示最新消息） |
| 应用在后台，频道保持 | 一条低优先级的常驻通知（Android 对前台服务的强制要求），静音、不显示角标 |
| 用户主动操作失败 | 仍然弹 Toast（例如未连接就发送、没有可分享的文件） |

两条通知分别走两个通知渠道，可以在系统设置里单独关掉其中一类：

```
easytalk-receive    接收通道     IMPORTANCE_LOW       常驻，静音
easytalk-messages   新消息       IMPORTANCE_DEFAULT   有声音，同一条被后续消息更新
```

**关于「后台接收」开关**：打开后，只要频道处于监听/连接状态，应用就会用前台服务把蓝牙
socket 保住，应用切到后台也不会断线；关掉则退到后台即断开（和以前一样）。
开关打开时会顺便申请通知权限（Android 13+ 必需）。

---

## 5. 自定义聊天背景（可选任务）

设置页 →「聊天背景」：

- **选择图片** → 走系统图片选择器（`ACTION_GET_CONTENT`，Android 4.0 起可用）
- **移除背景** → 恢复到纯色

实现细节，都是为了老机器不炸：

- 选中的图片先拷进应用私有目录，**两遍解码**：先只读尺寸，再按屏幕大小取 2 的幂次 `inSampleSize` 解码
- 解码用 `RGB_565`（每像素 2 字节，而不是 4 字节），并捕获 `OutOfMemoryError`
- 存储为 JPEG q88，一张全屏背景通常几十 KB
- 图片上盖一层当前主题背景色的 **60% 蒙版**，所以浅色主题配浅色文字、暗色主题配浅色文字都清楚
- 有背景时消息气泡会略微透明（约 91%）让图片透出来，同时不影响阅读
- 应用本身依旧**不含任何图片资源**：这张图完全来自用户运行时选择

---

## 6. 界面结构

```
┌──────────────────────────────────────────────┐
│ 蓝易聊                   设备      设置       │  ← 顶栏：纯文字按钮
├──────────────────────────────────────────────┤
│ 空闲 · 未监听                                 │  ← 状态行
├──────────────────────────────────────────────┤
│ ┌──────────────────────┐                     │
│ │ 12:04   小明          │                     │  ← 收到的：靠左色块
│ │ 晚上七点在地铁站见      │                     │
│ └──────────────────────┘                     │
│                     ┌──────────────────────┐ │
│                     │ 12:05   我            │ │  ← 发出的：靠右色块
│                     │ 好的，我提前十分钟到    │ │
│                     └──────────────────────┘ │
├──────────────────────────────────────────────┤
│ 输入消息…                            发送     │
└──────────────────────────────────────────────┘
```

- 所有「按钮」都是 `TextView`，靠强调色和字重区分，没有图标、没有边框、没有阴影
- 消息块是代码里生成的 `GradientDrawable` 纯色圆角矩形，**整个 APK 里没有一个 drawable 资源文件**
- 配色的 8 个色槽由主题属性（`values/attrs.xml`）提供，切换主题只需 `setTheme()` + `recreate()`

### 主题

`AppTheme.Light` / `AppTheme.Dark` 两套扁平色板，设置页三选一（暗色 / 浅色 / 跟随系统）。
因为不用 AppCompat，暗色模式是自己实现的，因此 **Android 4.0 上同样生效**。

### 语言

`values/`（英文，默认）、`values-zh/`、`values-ja/`、`values-ru/`，各 92 条字符串，键完全对齐。
应用内切换语言用 `Resources.updateConfiguration`（API 14→34 一致可用），切完
`recreate()`；底层的聊天页通过 `App.configVersion` 计数在 `onResume` 时感知并重建。

---

## 7. 从源码构建

不需要 Android Studio，也不需要 Gradle。

```powershell
# 1) 一次性下载工具链（约 310 MB，放进 .tools/）
pwsh -File tools/fetch-tools.ps1

# 2) 出包
pwsh -File build.ps1 -Clean
#    → releases/v0.0.11/bluetooth-easy-talk-0.0.11.apk
```

`build.ps1` 就是 AAPT2 → javac → d8 → 打包 → zipalign → apksigner 这条经典流水线：

```
aapt2 compile --dir res -o build/res.zip
aapt2 link -o build/app-unsigned.apk -I android.jar --manifest AndroidManifest.xml ... build/res.zip
javac -source 8 -target 8 -bootclasspath android.jar -d build/classes <sources> <gen/R.java>
d8 --min-api 14 --lib android.jar --output build/dex <classes>
   → classes.dex 塞进 APK → zipalign -f -p 4 → apksigner sign (v1 + v2)
```

版本号在 `build.ps1` 顶部的 `$versionName` / `$versionCode`，输出目录自动跟着版本走。

工具链已放在仓库内的 `.tools/x/`（JDK 17 / build-tools 34 / platform 34），
也可以用 `JAVA_HOME`、`ANDROID_HOME` 指向你自己的安装。

> `app/build.gradle`、`settings.gradle` 只是方便用 Android Studio 打开工程，
> **本次没有执行验证过**；已验证的构建路径是 `build.ps1`。

---

## 8. 测试

```powershell
pwsh -File tools/run-tests.ps1
```

`tools/CodecTest.java` 在桌面 JVM 上直接编译并运行 `TxtCodec.java`
（**编进 APK 的就是同一个源文件**），当前 **70 项断言全部通过**，覆盖：

- 帧头的生成与解析；畸形帧头（缺字节数、负数、空文件名、非数字）必须被拒绝
- 零长度正文（空的 `.txt` 也是一条消息）
- 文件名里含空格、发送者名里含下划线
- 中文 / 日文 / 俄文 / 带音符拉丁字母的发送者名，文件名往返一致且不超过 255 字节
- 路径穿越（`../../secret.txt`）、Windows 非法字符、控制字符、纯点号名
- 超长帧头与截断帧体必须报错
- **完整链路回环**：按发送端写法拼出一串帧，再按接收端写法读回来，
  逐条逐字节比对，其中包含中日俄文、表情符号、多行正文，
  以及一条正文本身就冒充帧头的消息

---

## 9. 项目结构

```
app/src/main/
├── AndroidManifest.xml
├── java/io/txtbt/chat/
│   ├── TxtCodec.java         线格式 + 文件命名（无 android 依赖，可单测）
│   ├── ChatStore.java        inbox/ outbox/ 的读写、导入、导出
│   ├── ChatEngine.java       RFCOMM 监听/连接/收发线程，单例
│   ├── TxtMessage.java       一条消息 = 一个 .txt 文件
│   ├── ChatBackground.java   自定义聊天背景：导入、降采样、缓存、蒙版
│   ├── TxtFileProvider.java  自研 FileProvider（API 24+ 分享 .txt 需要）
│   ├── ChatService.java      前台服务 + 两个通知渠道
│   ├── MainActivity.java     聊天页
│   ├── DevicesActivity.java  设备/频道页
│   ├── SettingsActivity.java 设置页
│   ├── App.java              前后台判定 + 前台服务策略
│   └── Skin.java  Lang.java  Prefs.java  Perm.java  Ui.java
└── res/
    ├── values/{attrs,styles,colors,strings}.xml   英文 + 主题色板
    ├── values-{zh,ja,ru}/strings.xml
    └── layout/{activity_main,activity_settings,activity_devices,row,section}.xml
tools/{CodecTest.java,run-tests.ps1,fetch-tools.ps1}
build.ps1
releases/v0.0.11/bluetooth-easy-talk-0.0.11.apk
```

---

## 10. 兼容性细节

| 项目 | 处理方式 |
|---|---|
| `minSdk 14`（Android 4.0） | 不用 `View.setBackground`(16)、`setShowWhen`(17)、`paddingStart`(17)、`Locale.forLanguageTag`(21)、`build()`(16，用 `getNotification()` 兜底)、Java 8 的集合/流 API |
| 老机器内存 | 背景图 `RGB_565` + `inSampleSize` 降采样 + 捕获 `OutOfMemoryError`；消息列表只渲染最近 200 条 |
| Android 12+ 蓝牙权限 | 声明并在运行时申请 `BLUETOOTH_CONNECT` / `BLUETOOTH_SCAN`；旧版本用 `BLUETOOTH` / `BLUETOOTH_ADMIN`（`maxSdkVersion=30`） |
| 存储权限 | 只用应用专属目录，因此 `WRITE_EXTERNAL_STORAGE` 带 `maxSdkVersion=18`，**新机型零存储权限** |
| Android 7+ 分享文件 | `TxtFileProvider` 提供 `content://` URI，绕开 `FileUriExposedException` |
| Android 8+ 后台接收 | `ChatService` 用 `startForeground` + 通知渠道；通知图标复用系统自带的 `android.R.drawable.stat_notify_chat`，所以应用本身依旧零图标资源 |
| Android 12+ 前台服务限制 | 前台服务在频道建立时（界面仍可见）就启动，避免"后台启动前台服务"被系统拒绝；被拒时静默降级不崩溃 |
| Android 13+ 通知 | 用户打开「后台接收」时才申请 `POST_NOTIFICATIONS`，不在启动时反复打扰 |
| 签名 | v1 + v2 + v3 全开；**v1 是 Android 4.x 能装的关键** |
| dex | `d8 --min-api 14` 产出 dex 035，Android 4.0 可直接执行 |

---

## 11. 已知限制（实话实说）

1. **实时聊天需要两端都装本应用。** 因为 Android 不允许第三方应用监听系统蓝牙
   （OBEX OPP）的收件目录，所以「对方没装应用」时只能走第 12 节的分享/导入流程，
   消息不会自动出现在对话里。
2. **必须先系统配对。** 应用不做配对流程，用系统的蓝牙设置配一次即可。
3. **一次只连一个对端**，不支持群聊；一方断开后可再点对方名字重连。
4. **消息时间以本机写入文件的时间为准**（`File.lastModified()`），
   因此两台手机时钟不一致时排序仍正确，但显示的时间戳会各自不同。
5. **后台通知依赖「后台接收」开关。** 关闭时应用退到后台即断开，自然也不会收到通知；
   这是该开关的字面含义，不是 bug。
6. **自定义背景只影响消息区**，顶栏、状态行和输入栏保持主题纯色。
7. 单条消息上限 256 KB；这是文本聊天，不是文件传输工具。
8. 调试包用固定口令 `android` 的自签名证书签名，仅供自用；
   上架或长期分发请换成你自己的 keystore。

---

## 12. 三种用法

| 场景 | 做法 | 对方需要装本应用吗 |
|---|---|---|
| **实时聊天** | 两端都装本应用；一端点「开始监听」，另一端在设备列表点对方名字 | 需要 |
| **把 .txt 发给任何人** | 设置页 →「用蓝牙分享 .txt」→ 系统分享面板选蓝牙（OBEX OPP） | **不需要** |
| **收下别人发来的 .txt** | 系统蓝牙保存文件后，在通知/文件管理器里「分享到 蓝易聊」 | **不需要** |

---

## 13. 许可

仅供学习与个人使用，随意修改。
