# termux-a11 修复记录（2026-10-24）

仓库：https://github.com/Zhiyu722/termux-a11 （fork 自 termux/termux-app）
本地工作副本：`/root/termux-app`
当前 `master`：`e5da3682`
版本：`versionCode 1024` / `versionName 0.119.0-beta.3-a11.1`
Release：https://github.com/Zhiyu722/termux-a11/releases/tag/v0.119.0-beta.3-a11.1
状态：**已验证用户在真机上装好，`/sdcard` 访问正常**

---

## 一、需求（来自 `/storage/emulated/0/MT2/apks/target.txt`）

1. 把仓库更新到「0.019」（= 用户在用的 `Termux_0.119.0-beta.3.apk`）的新功能，并**保留 A11 存储修复**。
2. 默认 apt 源改清华 TUNA（指 **Termux 内部**的 `$PREFIX/etc/apt/sources.list`，不是本机 Debian 的源）。
3. push 到 GitHub 并用 Action 编译。
4. 新增「保持屏幕常亮」功能。

---

## 二、关键事实与判断

### 1. 用户给的 APK 就是官方 v0.119.0-beta.3，不是魔改版
`aapt2 dump badging` → `versionCode='1022'`、`versionName='0.119.0-beta.3'`、AGP 4.2.2、compileSdk 30、minSdk 24。
和 `v0.119.0-beta.3` tag 源码的 `build.gradle`/manifest 完全对得上（初判「第三方魔改」是错的，已推翻）。

### 2. 仓库基线其实是官方 master，比 beta.3 更新
compare `v0.119.0-beta.3...master` → diverged，ahead 69 / behind 45。
beta.3 独有那 45 个提交里，绝大多数是 Release 提交和「退回旧行为」的兼容补丁（旧 `TERMUX_APP__*` 变量名、禁用 `TERMUX_APPS_DIR`/`TermuxAmSocketServer` 等）。
**真正可移植的功能只有 3 项**，已全部补上：

| 上游提交 | 内容 | 落在哪 |
|---|---|---|
| `8cdeb552` | Android 11+ 优先申请 `MANAGE_EXTERNAL_STORAGE` | `PermissionUtils.java` + `TermuxActivity.java` |
| `2cfbfcd7` | 默认扩展键改 `DRAWER`/`PASTE`/`SCROLL` | `TermuxPropertyConstants.java`（1 行） |
| `46cd5143` | bootstrap 变体 min/max SDK 校验 | `app/build.gradle` + `TermuxInstaller.java` + `strings.xml` |

所以**没有把仓库 rebase 到 beta.3**（那等于降级、丢掉 master 的 69 个修复），而是「版本号对齐 beta.3 语义 + 精准移植缺失功能」。

### 3. 最大的坑：存储权限移植写错了，`ls /sdcard` 仍然 Permission denied
第一版移植照抄上游：`checkLegacyStoragePermission = SDK_INT < 30`，且申请时 `prioritize = true`。后果：

- Android 11–13 上只检查 `Environment.isExternalStorageManager()`（「所有文件访问」AppOp）；
- 申请时又被 `prioritize=true` 直接踢到「所有文件访问」设置页；
- **`READ/WRITE_EXTERNAL_STORAGE` 运行时权限从未被申请过**，进程 UID 没有存储权限 → `ls /sdcard` → `cannot open directory '.': Permission denied`。

正确逻辑（第二版，也就是最终版）：

- 检查：`hasStorageAccessPermission()` —— 先看 `isExternalStorageManager()`；否则用 legacy 权限判断；
  Android 14+ 还要 `Environment.getExternalStorageDirectory().list()` 实测能否列目录（因为那边 legacy 只给媒体粒度权限）。
- 申请：`shouldPrioritizeManageExternalStoragePermission() = SDK_INT >= 34`。
  - Android 11–13 → 走 legacy 权限，普通授权弹框，点了就生效；
  - Android 14+ → 走「所有文件访问」。

教训：**上游的 `prioritize=true` 只适用于它自己的构建配置场景；本仓库 targetSdk=28 时直接照抄会导致 legacy 权限永不申请。** 移植涉及权限的代码必须对照 targetSdk 想清楚。

---

## 三、代码改动清单

| 文件 | 改动 |
|---|---|
| `app/build.gradle` | `versionCode 1024`、`versionName "0.119.0-beta.3-a11.1"`；新增 `bootstrapMinSdk/MinRelease/MaxSdk/MaxRelease`、`buildMinSdk/buildTargetSdk`；4 个 `buildConfigField`；`minSdk` 按变体（android-7 → 24，android-5 → 21） |
| `app/src/main/java/com/termux/app/TermuxActivity.java` | 首次启动自动申请存储权限；`hasStorageAccessPermission()`；`shouldPrioritizeManageExternalStoragePermission()`；`requestStoragePermission()` 传该判断；补 `import android.os.Build/Environment`、`java.io.File` |
| `app/src/main/java/com/termux/app/TermuxInstaller.java` | bootstrap 装完写 `$PREFIX/etc/apt/sources.list` 为清华 TUNA（`writeDefaultAptSourcesList()`）；新增 `checkIfMinOrMaxSdkVersionIsIncompatible()` 及其调用 |
| `termux-shared/.../PermissionUtils.java` | 新增 `prioritizeManageExternalStoragePermission` 参数（`checkAndRequestLegacyOrManageExternalStoragePermission`、`...IfPathOnPrimaryExternalStorage`）+ javadoc |
| `termux-shared/.../TermuxPropertyConstants.java` | 默认扩展键 → `[['ESC',{key:'DRAWER',popup:'PASTE'},'SCROLL','HOME','UP','END','PGUP'], [...]]` |
| `app/src/main/res/xml/termux_terminal_view_preferences.xml` | 新增 `keep_screen_on` 开关 |
| `app/.../settings/termux/TerminalViewPreferencesFragment.java` | `keep_screen_on` 的 get/set（映射到既有偏好 `screen_always_on`） |
| `app/.../terminal/TermuxTerminalViewClient.java` | `onResume()` 重新应用 `setKeepScreenOn()`，改完设置回终端立即生效 |
| `app/src/main/res/values/strings.xml` | keep-screen-on 三件套 + bootstrap 变体不兼容提示两条 |
| `.github/workflows/debug_build.yml` | release 标签 `TAG="v${VER}"`（原来多拼一次 `-a11`，产生过 `v0.119.0-beta.3-a11-a11`） |
| `README.md` / `CHANGELOG-a11.md` | 定制说明与变更记录 |

---

## 四、环境坑（下次省时间）

1. **直连 github.com 完全不通**：`curl https://github.com/...` 超时、`git clone/fetch/push` 报
   `GnuTLS recv error (-110)`。但 `api.github.com` 和 `gh-proxy.com` 可用。
   - 拉源码/下 APK：`https://gh-proxy.com/https://github.com/...`（tarball、release asset 都行）
   - 元数据：`gh api`
   - **push**：git 推不上去（gh-proxy 也不支持带认证的 push），改用 Git Data API 建 commit 再 PATCH ref。
     脚本已存 `/root/push_api.py`（自动读 `gh auth token`，按 `HEAD^..HEAD` 的改动建 blob → tree → commit → 更新 master）。
2. **本地不能编译**：没有 Android SDK（`ANDROID_HOME` 空）、没有 gradle/zipalign/apksigner，只能靠 GitHub Actions 验证。
3. **代理下大文件慢**：38MB 的 APK 走 gh-proxy 常有 280s 超时，超过就后台下。
4. `gh api --jq` 里不能写 `{status,ahead:ahead_by}` 这种（会当成函数调用），要写 `.status, .ahead_by`。

---

## 五、验证手段（可复用）

```bash
# 版本/权限
aapt2 dump badging app.apk | head -2
aapt2 dump permissions app.apk | grep -i "EXTERNAL\|MANAGE"

# 确认改的代码真的进了 dex
unzip -o -q app.apk -d x && for d in x/classes*.dex; do
  echo "$d tuna=$(strings $d | grep -c mirrors.tuna) keep=$(strings $d | grep -c keep_screen_on)"; done
```

实测结果：`versionCode='1024'`、`MANAGE_EXTERNAL_STORAGE` 在清单里、dex 中能搜到 `mirrors.tuna.tsinghua.edu.cn` 和 `keep_screen_on`。

---

## 六、遗留 / 下次可做

- release 里还留着历史 tag `v0.119.0-beta.3-a11`（对应 `+d11358f` 那个有 bug 的包），没删；需要可清理。
- master 里 `gradle.properties` 的 `minSdkVersion=21` 现在 app 模块不再直接用（app 用 `buildMinSdk`），其余模块仍引用，别乱删。
- 若要彻底对齐上游最新版，只需把 master 的 upstream 同步一遍，上述 4 项定制都在少数几个文件里，冲突面很小。
