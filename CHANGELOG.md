# Changelog

## 1.0.0

First public release. Supports Quark and Quark Netdisk 7.4.9.1045 only; other versions are refused.

### Default Actions

- Disable the updater services `QuarkUpdaterService1.0.0.21` and `QuarkUpdaterInternalService1.0.0.21`.
- Add an outbound firewall rule for every `updater.exe` under `%ProgramFiles%\QuarkUpdater\QuarkUpdater\<version>\`.
- Empty `%LOCALAPPDATA%\Quark\User Data\updates`, where `quark.exe` stages a full newer installer on its own, and deny the current user write access to it.
- Block the Explorer shell extension `{99D4E39A-EA7F-453E-9488-58C56BBF7B98}` (Quark AI Context Menu) and delete its handler keys.
- Delete static `Quark.*` verbs under `HKCU\Software\Classes\*\shell` and `Directory\shell` whenever they appear.
- Turn off the floating ball (`browser.quark.floating_ball.settings`) and the netdisk clipboard watcher (`clipboardForDownloadEnable`).
- Delete the native messaging host `com.quark.hostclient` for Chrome and Edge (HKCU and HKLM).

### Options

`-DisableUpdaterTasks`, `-BlockUpdaterSelfUpdate`, `-BlockAccelerator`, `-BlockDesktopWidget`, `-ReleaseTorrentMagnet`, `-KeepOffThisPC`, `-NoAutoLaunch`, `-NoBackgroundMode`, `-BlockAgentRuntime`, `-BlockHostClientManifest`.

### Modes

Dry run by default, `-Apply`, `-Install` (re-applies at every logon and every 6 hours through the task `QuarkDebloat-Watch`), `-Uninstall` (replays the journal in reverse).
