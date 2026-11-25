# 夸克是怎么把这些功能拉起来的

以下结论来自对夸克 7.4.9.1045 的静态分析（在 `quark.dll` 和网盘界面包 `app.asar` 里搜字符串、读相关代码），以及在一台装好并登录过的机器上的观察：进程和连接采样、注册表和文件的前后快照、管理员权限下导出的服务和计划任务、一次受监控的 20 分钟会话。路径、键名和服务名只对这个版本成立。

## 一句话版本

夸克浏览器和夸克网盘是同一个程序：Chromium 144 内核的 `quark.exe` 加 `quark.dll`，网盘是用 `--brand-clouddrive` 打开的一个标签页。功能开不开，主要看用户配置目录里的几个明文 JSON，和服务端下发的 `cms_config`。

会把东西装回来、改回去的有三条路：
1. 更新器 QuarkUpdater：手动启动的 SYSTEM 服务。夸克一启动就通过 COM 把它拉起来，它静默改写安装目录；另有计划任务定时唤醒它。
2. `quark.exe` 自带的更新下载器：不走更新器，直接把完整的新版安装包下到用户配置目录，再弹提示。
3. 网盘和 `quark.dll` 每次启动时重写的注册表项：.torrent 关联、magnet: 协议、开机自启。

脚本把前两条路堵住。第三条的几项夸克自己都有开关，脚本可以替你关掉，都是选项。整个过程不改二进制。

## 启动链

```
用户打开夸克或夸克网盘（桌面图标、开始菜单、开机自启 --launch-from=loginitem）
└─ quark.exe + <版本>\quark.dll（Chromium 144）                                    [核心]
   ├─ 网盘标签页（--brand-clouddrive），界面代码在 <版本>\Resources\app.asar          [核心]
   │    ├─ 启动时读 User Data\preference.json 里的开关
   │    ├─ onRegistryUpdate：按开关写入或删除 .torrent 关联和 magnet: 协议
   │    ├─ addToMyComputer：按开关把「夸克网盘」加进此电脑
   │    ├─ 剪贴板监听：登录后读每一次复制的文字，找分享链接和磁力链
   │    ├─ addFileContextMenu：能写 HKCU 下 *\shell、Directory\shell 的 Quark.<动作>
   │    └─ 桌面小组件 clouddrive_desktop_widget.exe：由界面或服务端开关拉起（推断）
   ├─ media_container.exe（启动时拉起，不显示窗口）                                     [核心]
   ├─ utility 进程：NetworkService（网络请求，网盘的上传下载也在这里）、
   │    NodeService ×3（AI agent，其中一个监听 127.0.0.1:9128）
   ├─ 悬浮球窗口：读 User Data\Default\Preferences 的 browser.quark.floating_ball.settings
   ├─ AI 运行时：启动后几分钟内装进 User Data\quark-agent\resources（约 1 GB）
   ├─ 原生消息宿主：Chrome、Edge 的 NativeMessagingHosts\com.quark.hostclient，代码里有检查并写入的逻辑（推断）
   ├─ 开机自启：每次启动重写 HKCU Run 下的 quark 值
   ├─ 程序内更新下载器：按服务端 upgrade_notify_policy，把新版完整安装包下到 User Data\updates
   ├─ cms_config：服务端配置（558 个键），定时拉取，每次整个覆盖
   └─ COM 激活 QuarkUpdater 服务（手动启动，以 SYSTEM 运行）
        └─ updater.exe
             ├─ quark_swap_util.exe --rename-chrome-exe：把暂存的 new_quark.exe 换成 quark.exe
             ├─ 删除旧的版本目录
             └─ 每 60 分钟检查一次组件补丁

计划任务 \QuarkUpdaterSystem\QuarkUpdater\QuarkUpdaterTaskSystem<版本>{GUID}（SYSTEM）
└─ updater.exe --wake --system → 同上

资源管理器
└─ Quark AI Context Menu {99D4E39A-EA7F-453E-9488-58C56BBF7B98}
     → %LOCALAPPDATA%\Programs\Common\Quark\quarkshellext_<时间戳>.dll

Chrome、Edge 里的夸克 AI 扩展（装了才有）
└─ quark_host_client.exe → quark.exe --launch-from=hostclient
```

几个位置：安装目录 `%ProgramFiles%\Quark`，用户配置 `%LOCALAPPDATA%\Quark\User Data`，更新器 `%ProgramFiles%\QuarkUpdater` 和 `%ProgramFiles(x86)%\QuarkUpdater`。版本目录和二进制在 `%ProgramFiles%\Quark\<版本>\`。

## 真正的开关

- **悬浮球**：`User Data\Default\Preferences` 里的 `browser.quark.floating_ball.settings`。夸克退出时改成 false，下次启动就没有悬浮球窗口；夸克运行中重写 Preferences，这个值也保持不变。界面里的「桌面悬浮球」开关写的就是它。
- **网盘的开关**：`User Data\preference.json` 的 `global:setting`。这是明文 JSON，不在 Chromium 的 Secure Preferences 签名保护范围里，网盘启动时读取：
  - `clipboardForDownloadEnable`：网盘的 `onClipboardChange` 读剪贴板之前先看它，为 false 就直接返回。
  - `registerUriSchemeSwitch.torrent` 和 `.magnet`：网盘启动时 `onRegistryUpdate` 按它们添加或删除 .torrent 关联和 magnet: 协议。
  - `permissionSwitch.setToMyComputerEnable`：为 true 时把「夸克网盘」加进此电脑（`{FB7670C2-7F99-426D-B687-21BB585A5C73}`），为 false 时删掉。
  - `permissionSwitch.autoLaunchEnable`：网盘的开机自启开关。`quark.dll` 每次启动都会重写 Run 值，代码里也有跳过重写的分支，但它看的是这个开关还是 StartupApproved（任务管理器「启动应用」里那个开关），还没确定。
- **后台运行**：`User Data\Local State` 的 `background_mode.enabled`，界面里「关闭夸克后继续运行后台应用」背后就是这个浏览器设置。
- **服务端配置**：`User Data\cms_config`，558 个键，每次拉取整个覆盖。推广弹窗、会员弹窗、更新提示的频率、默认播放器抢占、桌面小组件推送、快捷方式引导、假扩展按钮都在这里。本地改了会和服务端抢，留不住，脚本不碰。

脚本改这些 JSON 时，只在夸克没运行时动手，只改目标值那一段文本，文件其余部分逐字不变。夸克运行时会重写这些文件，运行中改了也会被盖掉。

夸克设置里的「开启系统菜单右键快捷功能」关掉时，只是在当前用户的挂载项 CLSID 后面加一个 `-`，系统级（HKLM）的挂载项照样有效。脚本改用 `Shell Extensions\Blocked`，资源管理器对所有用户都不再加载这个 CLSID。

## 为什么不改二进制

`quark.exe` 和 `quark.dll` 都有阿里巴巴的 Authenticode 签名，更新器还会整个替换 `quark.exe`。要的效果全都能通过服务、计划任务、防火墙、注册表、目录权限和夸克自己的开关做到，用不着打补丁。

## 回装路径和怎么堵

| 回装路径 | 谁来写、什么时候 | 脚本怎么堵 | 测试结果 |
|---|---|---|---|
| 更新器换掉 quark.exe、删旧版本目录、打组件补丁 | SYSTEM 服务，夸克启动时被 COM 激活；组件补丁每 60 分钟查一次 | 禁用两个服务；所有版本目录下的 `updater.exe` 加出站防火墙规则 | 以用户身份、用唤醒任务的参数启动更新器，几秒后退出，没有拉起服务，没有连接，没有改文件 |
| 唤醒计划任务 | SYSTEM 计划任务，运行 `updater.exe --wake --system` | `-DisableUpdaterTasks` 禁用 | 测试机上两个唤醒任务在脚本运行前就已是禁用，脚本没改过它们；禁用这一步本身没测过 |
| 服务端下发的完整安装包地址 | 更新器 | 同第一行 | 更新器目录里没有出现新安装包 |
| 程序内更新下载器 | `quark.exe` 以用户身份运行，按服务端的检查间隔 | 清空 `User Data\updates`，拒绝当前用户写入；目录被删了，下次检查时带着拒绝重建 | 安装后目录为空，写入测试被拒绝 |
| 浏览器原生消息宿主 | `quark.dll`，代码里有检查并写入的逻辑（推断） | 删除注册项；`-BlockHostClientManifest` 再换掉清单文件 | 删除后重新启动夸克，40 秒后注册项仍不在 |
| 静态右键菜单 `Quark.*` | 网盘界面代码，触发条件未知 | 出现就删 | 目前不存在，属于预防 |
| .torrent 关联 | 网盘每次启动按开关重写 | `-ReleaseTorrentMagnet` 关掉开关 | 受监控的会话里，看到夸克把被清空的关联写了回去 |
| 悬浮球开关 | 夸克或服务端 | 自动模式在夸克退出时改回 false | 手动改回 true 后跑一次检查，值恢复为 false，文件和改回前逐字节相同 |
| 用管理员身份运行官方安装包 | 你自己 | 挡不住 | 之后重跑一次 `-Install` |

一个手动启动的 SYSTEM 服务，任何用户进程都能通过 COM 把它拉起来。所以只禁用唤醒任务不够，服务本身也要禁用，防火墙规则是最后一层。

**旧版本的更新器可能被安全软件保护。** 装了一段时间的机器上，`QuarkUpdater\QuarkUpdater\` 下可能同时有新旧两个版本的更新器，各带一对服务和一个唤醒任务。测试时，旧版本那对服务改不了启动类型：管理员权限下 `sc config` 返回「拒绝访问」，系统日志里是服务控制管理器的 7006 事件（写 `Start` 值被拒），而服务本身的权限明明允许管理员修改。最可能的原因是机器上的安全软件在保护这几个服务，但没有证实。后来以用户身份唤醒这个旧版本时，它发现新版本已经在用，就把自己卸载了：删掉两个服务、唤醒任务和整个版本目录。所以脚本只按名字禁用当前版本 1.0.0.21 的两个服务；防火墙规则用通配符覆盖每个版本目录下的 `updater.exe`，唤醒任务也按通配符匹配。如果你的机器上还留着旧版本，又装了安全软件，旧服务可能同样改不动，这时挡住它联网的就是防火墙规则。

## 同名占位

几个选项用同名占位来挡：
- 程序文件：删掉 `xxx.exe`，在原位置建一个叫 `xxx.exe` 的空目录。之后任何程序用这个路径启动它都会失败，更新器想把文件写回来也会因为同名目录占着位置而失败。原文件先备份，撤销时校验哈希后放回。
- 目录：删掉整个目录，在原位置放一个同名的只读空文件，程序就没法再建出这个目录。AI 运行时（`-BlockAgentRuntime`）就是这样处理的；它有约 1 GB，不备份。

换掉 AI 运行时目录之前，脚本会检查里面的模块有没有被正在运行的进程加载，加载着就推迟，等下一次运行再做。

## 跨软件的劫持

除了夸克自己的进程，还查了它在别的软件里留下的东西：资源管理器、浏览器、Office、文件关联、剪贴板、网络和后台资源。

| 发现 | 证据 | 处理 |
|---|---|---|
| 资源管理器右键菜单扩展 `{99D4E39A-...}`（夸克 AI、上传到网盘、发送到手机、格式转换） | 已观察 | 默认拦截 |
| 网盘能写的静态右键菜单 `Quark.*`，能绕过上面的拦截 | 代码里有，目前没出现 | 默认，出现就删 |
| 「此电脑」里的夸克网盘入口 `{FB7670C2-...}` | 代码里有，开关关着时不出现 | 选项 `-KeepOffThisPC` |
| Chrome、Edge 的原生消息宿主 `com.quark.hostclient`，允许夸克自己的两个扩展（`nmaekpmealpjglikpijiegglabclhefp`、`enjndociabbicklhimdbcckdibpckfnm`）启动 `quark_host_client.exe` | 已观察；没装那两个扩展也照样注册 | 默认删除；选项 `-BlockHostClientManifest` |
| 默认视频播放器：安装时把 .mkv、.mov、.ts、.webm 改成夸克，服务端让它每 300 秒检查、最多抢回 3 次 | 已观察 | 没有本地开关，要你在 Windows 设置里改 |
| .torrent 和 magnet: 归夸克，每次启动重写 | 已观察 | 选项 `-ReleaseTorrentMagnet` |
| 注册为浏览器、PDF、图片、电子书等六十多种类型的候选程序；一旦被选为默认，就启用「保护默认」逻辑 | 已观察；没被选为默认时没有影响 | 不处理 |
| 剪贴板监听：登录后读每一次复制的文字 | 代码里有 | 默认关掉开关；浏览器那一侧的网址识别没有单独开关，保留 |
| Office 插件 DLL 放在 `%CommonProgramFiles%\Quark\drive`，没有注册 | 已观察 | 不处理 |
| HKCU Run 的 `quark` 值每次启动重写 | 已观察 | 选项 `-NoAutoLaunch`；任务管理器里的启动应用开关能兜底 |
| 关掉窗口后后台常驻，十几个进程 | 已观察 | 选项 `-NoBackgroundMode` |
| 桌面小组件能被服务端开关自动加进开机启动 | 代码里有 | 选项 `-BlockDesktopWidget` |
| 启动几分钟内静默装好约 1 GB 的 AI agent 运行时（Python 3.14、numpy、pandas、onnxruntime 等），即使 AI 工具栏是关着的 | 已观察 | 选项 `-BlockAgentRuntime` |
| AI 的 NodeService 在 127.0.0.1:9128 监听（浏览器自动化和 MCP 网关） | 已观察 | 在夸克进程里，只在本机回环，保留 |
| `quark.exe` 和更新器调用 Wi-Fi 定位 | 已观察 | 没有针对单个程序的可靠开关；可以关掉全机的桌面应用位置权限 |
| 服务端要求上报某些安全软件和电脑管家是否在运行 | 服务端配置里有 | 在夸克进程里，保留 |

没有发现的：图标覆盖、发送到、新建菜单、属性页这类外壳扩展，虚拟打印机，输入法，IE 扩展，AppInit_DLLs，Winsock LSP，P2P 或 PCDN 上传共享，UPnP 端口映射，厂商自己加的防火墙规则。夸克没运行时扫过所有能读的进程，没有夸克的 DLL 注入别的进程；夸克运行时的全进程扫描还没做。

## 证据等级

每个动作都标了证据等级，演练输出里也会打印：
- observed：在机器上直接看到。
- anchored：厂商的代码或配置里写明了，行为本身没有直接看到。
- inferred：从看到的事实推出来的。
- hypothesis：合理，但没测过。

只有 observed 和 anchored 的动作默认执行。inferred 和 hypothesis 的动作都做成了选项，演练时显示为 `[INACTIVE]`，并写明打开它的开关。也有几个 observed 和 anchored 的动作做成了选项：BT 接管和此电脑入口是取舍问题；唤醒任务在测试机上本来就是禁用的，禁用这一步还没真正跑过。

| 动作 | 默认/选项 | 证据 |
|---|---|---|
| 禁用更新器服务 | 默认 | observed |
| 更新器出站防火墙 | 默认 | observed |
| 拦截 AI 右键菜单 | 默认 | observed |
| 关悬浮球 | 默认 | observed |
| 清空并锁住 `updates` | 默认 | observed |
| 删除原生消息宿主注册项 | 默认 | observed |
| 关剪贴板监听 | 默认 | anchored |
| 删除静态右键菜单 `Quark.*` | 默认 | anchored |
| 禁用唤醒任务 | `-DisableUpdaterTasks` | observed（任务和它的动作在管理员权限下导出过；禁用这一步没测过） |
| 放弃 BT 和磁力接管 | `-ReleaseTorrentMagnet` | anchored |
| 此电脑入口 | `-KeepOffThisPC` | anchored |
| 开机自启开关 | `-NoAutoLaunch` | inferred |
| 后台运行开关 | `-NoBackgroundMode` | inferred |
| 替换宿主清单文件 | `-BlockHostClientManifest` | inferred |
| 去掉 AI 运行时 | `-BlockAgentRuntime` | hypothesis |
| 拦截 `QuarkUpdaterUpdate.exe` | `-BlockUpdaterSelfUpdate` | hypothesis |
| 占位加速器 | `-BlockAccelerator` | hypothesis |
| 占位桌面小组件 | `-BlockDesktopWidget` | hypothesis |

## 测过什么

下面这些在一台装了自动模式的机器上做过，结果都通过。前四项用的是较早的构建，当时还没有 `updates` 锁定、剪贴板开关、原生消息宿主和静态右键菜单这几项；劫持项那一条用的是加上它们之后的构建。
- **重启**：重启后登录时的那次检查记录 0 项改动，所有拦截都还在。同一次检查另记了 2 个失败，就是上文那对改不了启动类型的旧版服务。
- **改回测试**：夸克退出时手动把悬浮球开关改回 true，跑一次自动检查，值变回 false，文件和改回之前逐字节相同。
- **更新器唤醒**：用唤醒任务自己的参数，以用户身份启动当前版本的 `updater.exe`。它几秒后退出，没有拉起服务，没有网络连接，两个更新器目录都没有文件变化，更新日志没有变长。
- **20 分钟会话**：在监控下开着夸克网盘 20 分钟。没有任何更新器进程，安装目录和更新器目录没有文件变化；夸克中途重写了 Preferences，悬浮球仍然是关的；资源管理器里没有加载任何夸克模块；网络连接从启动时的 79 个降到 9 个左右，是启动时的一阵请求，不是失败后反复重试；网盘标签页登录正常，文件列表能显示。加速器、桌面小组件和 `QuarkUpdaterUpdate.exe` 在这期间都没有启动。
- **劫持项**：清空并锁住 `updates` 目录、关剪贴板监听、删四个原生消息宿主注册项，一次完成，没有失败。之后 `updates` 为空，写入测试被拒绝；剪贴板开关为 false；重新启动夸克 40 秒后，宿主注册项仍然不在。
- **自动检查**：改回测试那次检查记录 1 项改动（就是把值改回去）；更新器唤醒、20 分钟会话和劫持项之后的检查都是 0 项改动、0 个失败。

没测过的：
- 上传和下载没有自动测过，网盘只确认到能登录、文件列表能显示。加上 `updates` 锁定等几项之后，网盘文件列表还没有重新确认：那次启动只看到第 40 秒，网盘标签页当时还是空白。
- 有人点了夸克的更新提示会怎样。`updates` 被锁住后下载应该失败，但没有试。
- 选项里 inferred 和 hypothesis 的动作，都没有在真机上打开测过。唤醒任务的禁用也没真正跑过，原因见上文。
- 夸克运行时的全进程模块扫描。
- 账号同步会不会把 `preference.json` 里的开关改回去。就算改回去，下一次自动检查也会再改。

## 已知的限制

- 只支持 7.4.9.1045，其他版本拒绝执行。
- 更新被挡住后，Chromium 144 内核收不到安全更新。
- 改设置文件的几项只在夸克没运行时做。夸克默认关掉窗口后还常驻后台，所以使用中被改回的值，要等某次检查碰上夸克已退出才改回来。
- `updates` 目录上的拒绝只管往里写，不管删除整个目录（`User Data` 归你的账户所有）。如果夸克在两次检查之间删掉目录、重建、再下载，新安装包能落下来，下一次检查会再清空。
- 管理员 PowerShell 如果是用另一个管理员账户打开的，HKCU 和 `%LOCALAPPDATA%` 下的改动会落到那个账户。
- 旧版本的更新器服务可能被安全软件保护，改不了启动类型（见上文）。以后出现的新版本更新器，服务名不在脚本里，要靠防火墙规则和唤醒任务的通配符来挡。
- 用管理员身份运行夸克官方安装包，会撤销大部分改动，之后重跑一次 `-Install`。
- 服务端推送的窗口内内容、进程内遥测、本机 9128 监听、浏览器侧的剪贴板网址识别，都去不掉。
