# 变更说明（本仓库：Zhiyu722/termux-a11）

本仓库基于官方 [termux/termux-app](https://github.com/termux/termux-app) 的 `master` 分支（含 `v0.119.0-beta.3` 之后的全部上游修复），在其之上叠加下列定制改动。

- 版本：`versionCode 1023` / `versionName 0.119.0-beta.3-a11`
- 说明：`versionCode` 为 `1023`，高于官方 `v0.119.0-beta.3` 的 `1022`，因此可以直接覆盖安装到官方 beta 版之上（签名均为上游 `testkey` debug 签名）。

## 相对官方 v0.119.0-beta.3 / v0.118.0 的改动

### 1. 修复 Android 11+ 无法访问 `/sdcard`

`app/src/main/java/com/termux/app/TermuxActivity.java`

- 首次启动（`savedInstanceState == null`）时检测存储权限，缺失则自动调用 `requestStoragePermission(false)` 申请，不再需要用户手动跑 `termux-setup-storage`。
- Android 11+ 检测/申请的是「所有文件访问权限」`MANAGE_EXTERNAL_STORAGE`，而不是旧的 `READ/WRITE_EXTERNAL_STORAGE`。

### 2. 移植 v0.119.0-beta.3 的存储权限新逻辑（上游 8cdeb552）

`termux-shared/src/main/java/com/termux/shared/android/PermissionUtils.java`

- `checkAndRequestLegacyOrManageExternalStoragePermission()` 与
  `checkAndRequestLegacyOrManageExternalStoragePermissionIfPathOnPrimaryExternalStorage()`
  新增 `prioritizeManageExternalStoragePermission` 参数。
- 该参数为 `true` 且系统为 Android 11+ 时，强制走 `MANAGE_EXTERNAL_STORAGE` 分支。
- 修复点：Android 14 上 `targetSdkVersion=28` 时，申请 legacy `WRITE_EXTERNAL_STORAGE` 只会弹出
  「照片/音乐/视频」细粒度授权，实际上拿不到 `/sdcard` 的文件读写；同时 `MANAGE_EXTERNAL_STORAGE`
  还能授权 Android 12+ 上 USB OTG 等可移动卷（`/mnt/media_rw/XXXX-XXXX`）。
- `TermuxActivity.requestStoragePermission()` 调用处传 `true`，即 Android 11+ 优先申请「所有文件访问」。

### 3. 默认 apt 源改为清华 TUNA 镜像

`app/src/main/java/com/termux/app/TermuxInstaller.java`

- bootstrap 解包完成、`$PREFIX` 就绪后，写入 `$PREFIX/etc/apt/sources.list`：

  ```
  deb https://mirrors.tuna.tsinghua.edu.cn/termux/apt/termux-main stable main
  ```

- 仅在全新安装 / 重装 bootstrap 时写入；用户之后用 `termux-change-repo` 或手工编辑不会被覆盖。

### 4. 新增「保持屏幕常亮」设置项

- `app/src/main/res/xml/termux_terminal_view_preferences.xml`：设置 → Termux → 终端视图，新增
  *Keep Screen On* 开关。
- `app/src/main/java/com/termux/app/fragments/settings/termux/TerminalViewPreferencesFragment`（同文件的
  `TerminalViewPreferencesDataStore`）：读写已有偏好 `screen_always_on`。
- `app/src/main/java/com/termux/app/terminal/TermuxTerminalViewClient.java`：`onResume()` 时重新应用该设置，改完开关回到终端立即生效。
- 原有的长按终端 → 右键菜单里的 *Keep screen on* 开关仍然可用，两者共用同一偏好。

### 5. 版本号

`app/build.gradle`：`versionCode 1023` / `versionName "0.119.0-beta.3-a11"`。

## 已包含的上游新版能力（`v0.119.0-beta.3` 及之后）

- 存储权限迁移到 `MANAGE_EXTERNAL_STORAGE`（见上）。
- bootstrap 第二阶段脚本（`termux-bootstrap-second-stage.sh`）、bootstrap 打包为 `libtermux-bootstrap.so`。
- Android 16 / Android 16 QPR1 兼容（`hiddenapibypass` 6.1）、`serialVersionUID` 崩溃修复。
- 终端 Sixel 图形支持、CSI 窗口/单元格像素尺寸上报、`Share selected text`、AutoFill 改进。
- 工具链升级：AGP 8.13.2 / Gradle 9.2.1 / compileSdk 36 / NDK 29。

## 构建

推送到 `master` 自动触发 `.github/workflows/debug_build.yml`，产物：

- `termux-app_v0.119.0-beta.3-a11+<sha>-apt-android-7-github-debug_{arm64-v8a,armeabi-v7a,x86,x86_64,universal}.apk`
- `termux-app_v0.119.0-beta.3-a11+<sha>-apt-android-5-github-debug_*.apk`（Android 5/6 专用）

Android 7+ 设备请装 `apt-android-7` 的 `universal`（不确定架构时）或对应架构包；Android 5/6 装 `apt-android-5`。
