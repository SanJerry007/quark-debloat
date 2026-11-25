# quark-debloat

一个脚本给 Windows 版夸克和夸克网盘停掉后台更新，去掉 AI 右键菜单、悬浮球和剪贴板监听，只留浏览和网盘。 Debloat Quark browser and Quark Netdisk 7.4.9.1045 on Windows: no silent updates, no AI context menu, no floating ball, no binary patching.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform: Windows](https://img.shields.io/badge/Platform-Windows%2010%20%2F%2011-blue?style=flat)](#用法)
[![PowerShell 5.1+](https://img.shields.io/badge/PowerShell-5.1%2B-blue?style=flat)](#用法)
[![Tested: Quark 7.4.9.1045](https://img.shields.io/badge/Tested-Quark%207.4.9.1045-green?style=flat)](#版本支持)
[![No binary patching](https://img.shields.io/badge/Binary%20patching-none-green?style=flat)](docs/how-it-works.md)
[![Auto re-apply](https://img.shields.io/badge/Quark%20self--repair-auto%20re--apply-green?style=flat)](#自动重做)
[![Release](https://img.shields.io/github/v/release/SanJerry007/quark-debloat?style=flat)](https://github.com/SanJerry007/quark-debloat/releases)
[![Languages](https://img.shields.io/badge/Languages-%E4%B8%AD%E6%96%87%20%2F%20EN-blue?style=flat)](#english-summary)
[![Last commit](https://img.shields.io/github/last-commit/SanJerry007/quark-debloat?style=flat)](https://github.com/SanJerry007/quark-debloat/commits)
[![Stars](https://img.shields.io/github/stars/SanJerry007/quark-debloat?style=flat)](https://github.com/SanJerry007/quark-debloat/stargazers)

[中文](#quark-debloat) | [English](#english-summary)

---

给 Windows 版夸克做减法。夸克浏览器和夸克网盘桌面版是同一套程序，脚本跑一次两边都处理。它停掉以 SYSTEM 身份静默改写安装目录的更新器，清空并锁住夸克自己偷偷下载的新版安装包，去掉资源管理器里的夸克 AI 右键菜单、桌面悬浮球、剪贴板监听，以及夸克注册进 Chrome 和 Edge 的原生消息宿主。浏览、网盘文件列表、上传下载和登录都保留。

不改任何二进制。脚本只动服务的启动类型、防火墙规则、注册表、夸克自己的配置开关和几个目录的权限，每一步都先记日志，可以原样撤销。装成自动模式后，每次登录和每 6 小时检查一遍，夸克或服务端把东西改回去了，就再改回来。原理、启动链和测试记录见 [docs/how-it-works.md](docs/how-it-works.md)。

只支持 7.4.9.1045 这一个版本，其他版本会被拒绝，见[版本支持](#版本支持)。

## 会去掉什么

| 功能 | 怎么处理 | 默认/选项 |
|---|---|---|
| 后台更新器 QuarkUpdater（SYSTEM 服务，夸克一启动就被拉起，静默替换 `quark.exe`、删旧版本目录、定时打组件补丁） | 两个 1.0.0.21 服务改为禁用；`%ProgramFiles%\QuarkUpdater\QuarkUpdater\<版本>\updater.exe` 一律加出站防火墙规则 | 默认 |
| 更新器的唤醒计划任务 `\QuarkUpdaterSystem\...\QuarkUpdaterTaskSystem*` | 禁用 | `-DisableUpdaterTasks`（推荐） |
| 更新器自己的更新程序 `QuarkUpdaterUpdate.exe` | IFEO 按文件名拦截 | `-BlockUpdaterSelfUpdate` |
| 程序内更新下载器（绕过更新器，把四百多 MB 的新版安装包下到 `%LOCALAPPDATA%\Quark\User Data\updates`，再弹更新提示） | 清空这个目录，禁止当前用户往里写 | 默认 |
| 资源管理器右键菜单里的夸克 AI、上传到网盘、发送到手机、格式转换（Quark AI Context Menu） | CLSID 加入系统的 Shell Extensions 黑名单，删除菜单挂载项 | 默认 |
| 网盘写的静态右键菜单 `Quark.*`（能绕过上面的黑名单） | 出现就删 | 默认 |
| 桌面悬浮球 | 关掉它自己的开关 `floating_ball.settings` | 默认 |
| 剪贴板监听（登录后每复制一段文字，网盘都读一遍，找分享链接和磁力链） | 关掉网盘自己的开关 `clipboardForDownloadEnable` | 默认 |
| 注册进 Chrome 和 Edge 的原生消息宿主 `com.quark.hostclient`（让浏览器里的夸克 AI 扩展拉起夸克） | 删除 HKCU 和 HKLM 下的注册项 | 默认 |
| 同一个宿主的清单文件 | 换成同名空目录，注册项被写回也指不到东西 | `-BlockHostClientManifest` |
| 「此电脑」里的夸克网盘入口 | 关掉开关，删除命名空间注册项 | `-KeepOffThisPC` |
| 开机自启 | 关掉网盘的开机自启开关 | `-NoAutoLaunch` |
| 关掉窗口后仍在后台运行 | 关掉浏览器的后台运行开关 | `-NoBackgroundMode` |
| 接管 .torrent 文件和 magnet: 链接 | 关掉两个接管开关，夸克下次启动时自己删掉关联 | `-ReleaseTorrentMagnet` |
| 约 1 GB 的 AI agent 运行时（Python、Node 和一批 pip 包，不用 AI 功能也会装） | 整个目录换成只读的同名空文件 | `-BlockAgentRuntime` |
| 第三方游戏加速器（带虚拟网卡驱动和证书工具） | 两个 exe 换成同名空目录 | `-BlockAccelerator` |
| 桌面小组件 | 两个 exe 换成同名空目录 | `-BlockDesktopWidget` |

默认项都有直接观察或厂商代码作证据。做成选项的有两种情况：一种是取舍，看你要不要（BT 接管、此电脑入口）；另一种是证据还不够，比如没观察到谁启动加速器，就默认不动，后台运行、开机自启的开关是否真的管用也还没验证。唤醒任务是例外：任务本身已经看到了，但测试机上它本来就是禁用的，禁用这一步没真正跑过，所以也留作选项，推荐加上。证据等级见 [docs/how-it-works.md](docs/how-it-works.md#证据等级)。

## 会保留什么

- 浏览器本体和夸克网盘：`quark.exe`、`quark_proxy.exe`、`quark.dll`、`app.asar`。文件列表、上传下载、同步和登录状态都不动。脚本里写死了这份核心文件清单，任何动作碰到它们都会被拒绝。
- `media_container.exe`：夸克一启动就拉起它，播放和截图要用。
- 问AI、agent、NodeService 这些 AI 组件：它们是夸克主程序的子进程，没有能单独关掉的开关。只有那 1 GB 运行时可以用 `-BlockAgentRuntime` 去掉。
- 进程内的遥测模块（`UTForPC.dll`、`Report.dll` 等）：加载在夸克自己的进程里，拆不开。
- Office 插件 DLL：只是放在 `%CommonProgramFiles%\Quark\drive`，没有注册进 Office，不会加载。

代价：
- 更新器和程序内下载器都被挡住后，夸克的 Chromium 144 内核收不到安全更新。要更新，见[版本支持](#版本支持)。
- 资源管理器右键里不再有「上传到夸克网盘」，要在网盘里点上传。
- 加了 `-ReleaseTorrentMagnet` 时，如果夸克是唯一的 magnet: 处理程序，点磁力链接会没有反应，直到别的软件接手。

## 挡不住什么

**服务端推送的窗口内广告。** 网盘首页的推广弹窗、会员弹窗、更新后打开的「What's new」推广页、工具栏上假的「夸克搜题」扩展按钮、桌面小组件和快捷方式的引导，都由服务端配置 `cms_config`（558 个键）控制。这份文件每次拉取都会被服务端整个覆盖，本地改了也留不住，所以脚本不碰它。

**要你自己动手的设置。** 夸克自己的开关大多存在明文 JSON 里，脚本能直接改。下面这几项改不了，或者应该由你决定：

| 设置 | 为什么脚本不改 | 在哪里改 |
|---|---|---|
| 默认视频播放器（安装时 .mkv、.mov、.ts、.webm 被改成夸克） | Windows 用哈希保护这个选择（UserChoice），脚本写不进去；夸克的服务端配置让它每 300 秒检查一次，最多抢回 3 次 | Windows 设置 > 应用 > 默认应用，按文件类型改回你用的播放器 |
| 系统右键菜单总开关 | 脚本已经在资源管理器这一层拦住了菜单，这个开关是夸克自己的关法，可选 | 夸克网盘 > 设置 > 通用 > 本地右键菜单设置 > 开启系统菜单右键快捷功能 |
| 关闭夸克后继续运行后台应用 | 关了以后窗口一关上传下载就停，由你决定；也可以加 `-NoBackgroundMode` | 夸克网盘 > 设置 > 通用 > 权限设置 |
| 接管 BT 种子和磁力链 | 由你决定；也可以加 `-ReleaseTorrentMagnet` | 夸克网盘 > 设置 > 通用 > 接管设置 |
| 桌面应用的位置权限（夸克和它的更新器都调用过 Wi-Fi 定位） | 这是全机开关，会影响所有桌面程序 | Windows 11：设置 > 隐私和安全性 > 位置 > 允许桌面应用访问你的位置；Windows 10：设置 > 隐私 > 位置 |

另外，夸克弹出「更新已就绪」之类的提示时，直接关掉，不要点更新。

**和夸克进程绑在一起的部分。** 进程内遥测（包括服务端要求上报的「是否在运行某些安全软件和电脑管家」）、AI 组件在 127.0.0.1:9128 的本机监听、浏览器那一侧的剪贴板网址识别。它们都在 `quark.dll` 里，没有单独的开关，防火墙也管不到本机回环。

## 用法

需要 Windows PowerShell 5.1 或更新的版本。`-Apply`、`-Install`、`-Uninstall` 要在管理员 PowerShell 里跑。从 [Releases](https://github.com/SanJerry007/quark-debloat/releases) 下载 `QuarkDebloat.ps1`，或者克隆本仓库。

动手前注意两件事：
1. **先完全退出夸克。** 关掉窗口不算，夸克默认会留在后台。从托盘图标选退出，或者在任务管理器里结束所有 `quark.exe`。几个设置文件只在夸克没运行时才改，夸克还开着时脚本会提示 WARN 并跳过这几项。
2. **在你自己的账户里打开管理员 PowerShell**（右键「以管理员身份运行」）。悬浮球、剪贴板、`updates` 目录这些改动落在当前账户的配置里。如果 UAC 要你输入另一个管理员账户的密码，这些改动会落到那个账户。

推荐直接装成自动模式（见[自动重做](#自动重做)）：

```powershell
# 精简一次，并保持下去（管理员 PowerShell，先退出夸克）
powershell -ExecutionPolicy Bypass -File .\QuarkDebloat.ps1 -Install -DisableUpdaterTasks
```

只想手动跑：

```powershell
# 先演练，只列出会改什么，什么都不改（不需要管理员）
powershell -ExecutionPolicy Bypass -File .\QuarkDebloat.ps1

# 真正执行一次（管理员 PowerShell）
powershell -ExecutionPolicy Bypass -File .\QuarkDebloat.ps1 -Apply

# 带选项执行
powershell -ExecutionPolicy Bypass -File .\QuarkDebloat.ps1 -Apply -DisableUpdaterTasks -NoBackgroundMode
```

演练会打印安装目录是怎么找到的、通过了哪项厂商身份校验（文件、签名、内容）、每一项计划改动，最后列出没选的选项各自会多做什么。脚本可以反复执行，已经处理过的项显示 `SKIP`。

退出码：0 正常；1 有一步失败，或者 `-Uninstall` 留下了没还原的项；2 在改动之前就拒绝了（模式冲突、没有管理员权限、没装夸克、身份校验没过、版本不支持）。

### 选项

| 开关 | 作用 |
|---|---|
| `-DisableUpdaterTasks` | 禁用更新器的唤醒计划任务 `QuarkUpdaterTaskSystem*`（在 `\QuarkUpdaterSystem\` 及其子目录下）。推荐加上。 |
| `-BlockUpdaterSelfUpdate` | 用 IFEO 按文件名拦截 `QuarkUpdaterUpdate.exe`（更新器自己的更新程序）。 |
| `-BlockAccelerator` | 把 `components\accelerator\` 下的 `accelerator.exe`、`old_accelerator.exe` 换成同名空目录。 |
| `-BlockDesktopWidget` | 把 `components\clouddrive_desktop_widget\` 下的两个 exe 换成同名空目录。 |
| `-ReleaseTorrentMagnet` | 关掉 BT 种子和磁力链接管，夸克下次启动时删掉自己的 .torrent 关联和 magnet: 处理程序。 |
| `-KeepOffThisPC` | 不让「此电脑」里出现夸克网盘入口，夸克写回就删。 |
| `-NoAutoLaunch` | 关掉网盘的开机自启开关。 |
| `-NoBackgroundMode` | 关掉最后一个窗口时让夸克退出。窗口一关，网盘的上传下载也会停。 |
| `-BlockAgentRuntime` | 把约 1 GB 的 AI agent 运行时目录换成只读空文件。不备份，撤销后夸克会重新下载；问AI 等功能可能失效。 |
| `-BlockHostClientManifest` | 把原生消息宿主的清单文件换成同名空目录（有备份），是删除注册项之外的第二层。 |

其他参数：

| 参数 | 作用 |
|---|---|
| `-InstallRoot <目录>` | 指定夸克安装目录，不自动查找。这个目录同样要通过厂商身份校验。 |
| `-Force` | 版本不在支持列表里也执行。不建议用。 |
| `-IgnoreUninspectable <进程名>` | 检查已加载模块时，跳过这些无法检查的进程。只对这一次运行有效。 |

## 自动重做

夸克会把一部分改动改回去：网盘每次启动按开关重写 .torrent 关联，服务端也可能把悬浮球打开；`quark.dll` 里还有检查并写入浏览器宿主注册项的代码（删掉后重启夸克，40 秒内没有写回）。`-Install` 就是为这个准备的：

1. 先执行一次 `-Apply`，没有失败项才继续。
2. 把脚本和这次用的选项复制到 `%ProgramData%\AppDebloat\quark`，这个目录只有管理员能改。
3. 注册计划任务 `QuarkDebloat-Watch`：每次登录时和每 6 小时，以你的身份、管理员权限、无窗口地跑一次 `-Auto`。

`-Auto` 只重新确认那些挡东西的动作。什么都没变时全部是 `SKIP`；有东西被改回去了，它就再改一次。完整输出在 `auto.log`，每次一行的摘要在 `history.log`，都在 `%ProgramData%\AppDebloat\quark`。如果夸克已经变成别的版本，`-Auto` 什么都不改，记一行 WARN，退出码 2。

改设置文件的几项（悬浮球、剪贴板和各个开关）只在夸克没运行时做。夸克关掉窗口后默认还留在后台，所以使用中被改回的值，要等到某次检查刚好碰上夸克已退出才会改回来。想缩短这个间隔，就从托盘退出夸克，或者加 `-NoBackgroundMode`。

脚本本身不会从网上自动更新，因为让一个管理员权限的任务执行网上下载的代码风险太大。新版本发布后，下载下来重新跑一次 `-Install` 即可。

```powershell
# 取消自动模式，并撤销脚本做过的所有改动（管理员 PowerShell，先退出夸克）
powershell -ExecutionPolicy Bypass -File .\QuarkDebloat.ps1 -Uninstall
```

装好后打开一次夸克网盘，确认能登录、文件列表能显示，再传一个小文件并下载回来，确认上传下载正常。

## 版本支持

只支持 7.4.9.1045。脚本从注册表的卸载项和 `quark.exe` 的文件版本读版本号，不在支持列表里时，`-Apply`、`-Install`、`-Auto` 都拒绝执行（退出码 2），演练照常能跑。这里的每一项都是对着这个版本查出来的：配置键名、版本目录、更新器服务名、组件路径，换个版本都可能变。不要用 `-Force` 在别的版本上硬跑，键名或路径一变，脚本可能报错，也可能什么都没挡住。

更新器和程序内下载器被挡住后，夸克会一直停在这个版本。想换版本：
1. 跑 `-Uninstall`，撤销所有改动。
2. 用夸克官方安装包或它自己的更新功能升级。
3. 用 [app-debloat](https://github.com/DaizeDong/app-debloat) 给新版本重新生成脚本。本仓库的脚本就是它生成的：它重新盘点安装、分析启动链和回装路径，再从一份 manifest 生成新的单文件脚本。

## 安全与撤销

- 不改任何二进制，不打补丁，不碰签名文件的内容。
- 每项改动都先记日志再动手。`%ProgramData%\AppDebloat\quark\journal.jsonl` 记下每一项改之前和改之后的状态。删除注册表项之前先导出到同目录的 `regkeys\`，替换厂商文件之前先备份到 `backup\`，还原时校验哈希。
- `-Uninstall` 按日志倒序还原。每一项只在当前状态仍是脚本写下的样子时才还原，否则报 `CONFLICT` 并保持原样，不会盖掉你后来自己做的改动。有 WARN、CONFLICT 或失败时，日志和状态目录都保留，处理完再跑一次。
- 有两样东西删了不备份：`updates` 目录里的新版安装包，和 `-BlockAgentRuntime` 去掉的 AI 运行时（约 1 GB）。它们都是夸克能重新下载的厂商文件，`-Uninstall` 结束时会把它们单独列出来，说明这些文件没有备份。
- 脚本只在几个允许的位置改文件：夸克安装目录、`%LOCALAPPDATA%\Quark\User Data` 下点名的几个文件和目录、两个更新器目录。系统目录、核心文件和这些位置之外的路径一律拒绝，路径上有 junction 之类的重解析点也拒绝。
- 不碰登录、会员、限速和付款相关的任何东西。

撤销步骤：
1. 完全退出夸克，在你自己账户的管理员 PowerShell 里跑 `-Uninstall`。它删掉计划任务和防火墙规则，恢复服务的启动类型，恢复删掉的注册表项、改过的开关和备份过的文件，去掉 `updates` 目录上的写入拒绝。全部还原干净后，`%ProgramData%\AppDebloat\quark` 也一起删掉。
2. 如果报了 `CONFLICT`，说明那一项在脚本之后又被改过。按提示手动处理，或者用夸克官方安装包覆盖安装。

## 常见问题

**夸克网盘的桌面悬浮球怎么关？** 它由 `User Data\Default\Preferences` 里的 `browser.quark.floating_ball.settings` 控制，界面位置是夸克网盘 > 设置 > 通用 > 权限设置 > 桌面悬浮球。脚本在夸克退出时把它设成 false；装成自动模式后，被改回去也会再关掉。

**夸克怎么禁止自动更新？** 夸克有两条更新路。一条是 SYSTEM 身份的更新器 QuarkUpdater：夸克一启动就通过 COM 拉起它的服务，静默替换 `quark.exe`。脚本禁用这两个服务，给 `updater.exe` 加出站防火墙规则，加 `-DisableUpdaterTasks` 再关掉唤醒任务。另一条在 `quark.exe` 自己身上：它把完整的新版安装包下到 `User Data\updates`，再弹更新提示。脚本清空这个目录，并禁止你的账户往里写。代价是收不到安全更新。

**`User Data\updates` 里那个几百 MB 的 exe 是什么，能删吗？** 是夸克自己下载的新版完整安装包，带阿里巴巴的数字签名。能删，脚本默认就清空这个目录并锁住。

**右键菜单里的夸克 AI、上传到夸克网盘怎么去掉？** 这个菜单来自 `quarkshellext.dll`，CLSID 是 `{99D4E39A-EA7F-453E-9488-58C56BBF7B98}`。夸克设置里的开关只是在当前用户下给挂载项加一个后缀，系统级的挂载项还在。脚本把这个 CLSID 加入 Shell Extensions 黑名单，资源管理器从此不加载它。

**网盘首页的推广弹窗和会员弹窗能去掉吗？** 不能。它们由服务端配置推送，本地那份配置每次都会被服务端覆盖。脚本只处理本地有开关、或者有固定落点的东西。

**为什么关掉窗口后夸克还在后台？** 夸克默认开着「关闭夸克后继续运行后台应用」，主窗口关掉后还有十几个 `quark.exe` 进程在跑。到夸克网盘 > 设置 > 通用 > 权限设置里关掉，或者加 `-NoBackgroundMode`。关掉后，窗口一关上传下载也会停。

**为什么 .mkv 之类的视频总被夸克打开？** 夸克安装时把 .mkv、.mov、.ts、.webm 的默认程序改成了自己，服务端配置还让它每 300 秒检查一次，最多抢回 3 次。Windows 用哈希保护这个选择，脚本改不了。到 Windows 设置 > 应用 > 默认应用里改回去；如果又被抢回，再改一次。

**夸克为什么在 Chrome 和 Edge 里注册了东西？** 那是原生消息宿主 `com.quark.hostclient`，让浏览器里的夸克 AI 扩展能启动夸克。没装那个扩展，注册项也照样在。脚本默认删掉它；加 `-BlockHostClientManifest` 再把清单文件换掉，注册项被写回也用不了。

**会不会把网盘弄坏，影响登录和上传下载？** 核心文件在脚本里受保护，任何动作碰到都会被拒绝；登录、会员、限速相关的东西一概不碰。测试时禁用更新器后，网盘照常登录、显示文件列表。先不加参数跑一次演练，看清会改什么再执行，所有改动都能用 `-Uninstall` 还原。

**夸克浏览器和夸克网盘要分别跑吗？** 不用。两者是同一套安装：同一个 `quark.exe`，网盘是用 `--brand-clouddrive` 打开的一个标签页，只是在「应用」列表里有两个卸载项。跑一次就都处理了。

搜索关键词：夸克 去广告、夸克网盘 精简、夸克 禁止自动更新、夸克 悬浮球 关闭、夸克 右键菜单 删除、夸克 后台运行、夸克 剪贴板 监听、QuarkUpdater 禁用、Quark debloat、Quark Netdisk remove updater、disable Quark auto update。

## 相关项目

- [sogou-ime-debloat](https://github.com/SanJerry007/sogou-ime-debloat)：同样的思路，处理 Windows 版搜狗输入法。
- [app-debloat](https://github.com/DaizeDong/app-debloat)：生成本仓库脚本的 skill。它给一个流氓 Windows 软件理清启动链、开关和回装路径，再从 manifest 生成单文件脚本。

## 说明

- 本项目与阿里巴巴、夸克无关。脚本只修改你自己电脑上已安装的软件。
- 仓库里没有任何账号信息、密钥或从别人电脑上拿来的数据。
- 仅供在自己的电脑上个人使用。
- 风险自负。

---

## English Summary

A PowerShell script that strips the self-updater, the in-app update downloader, the Explorer AI context menu, the floating ball, the clipboard watcher and the Chrome and Edge native messaging host from Quark browser and Quark Netdisk 7.4.9.1045 on Windows (both are one install), without patching any binary. By default it disables the services `QuarkUpdaterService1.0.0.21` and `QuarkUpdaterInternalService1.0.0.21`, adds an outbound firewall rule for every `updater.exe` under `%ProgramFiles%\QuarkUpdater\QuarkUpdater\<version>\`, empties `%LOCALAPPDATA%\Quark\User Data\updates` (where `quark.exe` stages a full newer installer on its own) and denies the user write access to it, blocks the shell extension `{99D4E39A-EA7F-453E-9488-58C56BBF7B98}`, deletes static `Quark.*` shell verbs, turns off Quark's own floating-ball and clipboard switches, and deletes `NativeMessagingHosts\com.quark.hostclient`. Options cover the updater wake tasks (`-DisableUpdaterTasks`, recommended), `QuarkUpdaterUpdate.exe`, the third-party accelerator, the desktop widget, .torrent and magnet: takeover, the This PC entry, autostart, background mode, the 1 GB AI agent runtime and the native host manifest.

Run `QuarkDebloat.ps1` for a dry run, then `-Install -DisableUpdaterTasks` (or a one-off `-Apply`) from an elevated PowerShell on your own account, with Quark fully closed (exit it from the tray icon). `-Install` copies the script to the admin-only `%ProgramData%\AppDebloat\quark` and registers the task `QuarkDebloat-Watch`, which re-applies the blocks at every logon and every 6 hours; `-Uninstall` replays the journal in reverse and restores each original only where the current state is still what the script wrote. Only version 7.4.9.1045 is supported; other versions are refused, and a new script can be generated with [app-debloat](https://github.com/DaizeDong/app-debloat). It cannot remove server-pushed in-window promos, the default video player takeover (change it in Windows Settings > Apps > Default apps) or in-process telemetry, and Quark's Chromium 144 engine gets no security updates while the updater is blocked. Not affiliated with Alibaba or Quark.
