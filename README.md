# Android 17 起 `pm uninstall --user 0` 无法卸载系统应用的根因与绕过

> 适用版本：Android 17（API 37）及以上。

---

## ⚠️ AI 生成声明

**本文由 AI 助手起草，实验在真实设备上执行，数据为实测记录。**

| 部分 | 来源 |
|---|---|
| 命令、错误文本、版本号、包状态字段 | **真机实测**（单一样本，见 E6），或直接引用原始来源 |
| 引用的源码补丁、提交信息、他人 issue 引述 | 来自文中给出的原始链接，**以原链接为准** |
| 根因分析、推论、章节组织、措辞 | **AI 生成**，可能存在错误或过度推断 |

请注意：

- 本文的**分析、推论与结论由 AI 生成**。作者已对照实测数据逐条核对，但**不保证正确**；尤其是第 3、4 节属于推断，请勿作为唯一依据
- 单机实测结果**不代表所有设备**（见第 6 节），请勿直接照搬到你的机器上
- 若本文与官方文档、源码或原始 issue 冲突，**一律以原始来源为准**
- 文中引用他人项目的结论时，若对方后续已更新或修正，请以对方最新状态为准，本文可能滞后
- 若某处结论看起来站不住脚，**大概率是本文错了**，而不是你错了

如果发现错误，欢迎指出；附上你的复现输出会更好核对，没有也完全欢迎。

---

## TL;DR

| 结论 | 说明 |
|---|---|
| `pm uninstall --user 0 <系统应用>` 在 Android 17 起被平台层拒绝 | 变更在 `PackageManagerShellCommand.runUninstall()`，**对所有厂商生效**，与 OEM 定制无关 |
| 部分 OEM 的 `pm disable-user` 同时被拦 | **另一条独立成因**：厂商在 `PackageManagerServiceImpl` 里加的，与 Android 版本无关 |
| 两条限制叠加，导致常见的非 root 去膨胀方法全部失效 | 看起来像"同一个限制在加码"，实际是四五个独立成因碰在一起（见 1.1） |
| 可行的非 root 绕过 | 以 shell 身份**绕过 `pm` 外壳**，直连 Binder 调 `IPackageManager.deletePackageAsUser`（现成工具：Shizuku + Canta） |
| 卸载状态 | 写入 `/data/system` 包状态，**跨重启持久**；恢复出厂会还原 |

---

## 1. 现象

在 Android 17 设备上对系统应用执行：

```
$ adb shell pm uninstall --user 0 <pkg>
Failure [only root can delete system app for a particular user]
```

加 `-k`（保留数据）后，报错完全相同：

```
$ adb shell pm uninstall -k --user 0 <pkg>
Failure [only root can delete system app for a particular user]
```

原因很直接：`-k` 只是追加 `DELETE_KEEP_DATA` 标志，而拦截发生在**参数解析之后、删除执行之前**，所以带不带 `-k` 结果一致。

### 1.1 命令变体矩阵（同一台 Android 17 设备实测）

| 命令 | 结果 | 拦截位置 |
|---|---|---|
| `pm uninstall --user 0 <系统应用>` | ❌ `only root can delete system app for a particular user` | `PackageManagerShellCommand.runUninstall()`（E1） |
| `pm uninstall -k --user 0 <系统应用>` | ❌ 同上 | 同上（`-k` 不影响） |
| `pm uninstall <系统应用>`（不带 `--user`） | ❌ `DELETE_FAILED_INTERNAL_ERROR` | 服务端：缺 `DELETE_SYSTEM_APP` 权限 |
| `pm disable-user --user 0 <系统应用>` | ❌ `SecurityException: Cannot disable system packages` | OEM 定制（见第 3 节下半） |
| `pm hide --user 0 <系统应用>` | ❌ `SecurityException: requires android.permission.MANAGE_USERS` | 权限层（shell 未申请该权限） |

`pm disable-user` 的完整异常：

```
java.lang.SecurityException: Cannot disable system packages.
    at com.android.server.pm.PackageManagerServiceImpl.shouldRestrictEnabledSettingsChange(PackageManagerServiceImpl.java:1116)
    at com.android.server.pm.PackageManagerService.setEnabledSettings(PackageManagerService.java:4530)
    at com.android.server.pm.PackageManagerService$IPackageManagerImpl.setApplicationEnabledSetting(PackageManagerService.java:6620)
    at com.android.server.pm.PackageManagerShellCommand.runSetEnabledSetting(PackageManagerShellCommand.java:2498)
```

> **上表每一行都是独立成因**：前两行是平台变更，第三行是权限不足，第四行是 OEM 策略，第五行是 AIDL 层的权限注解。看起来"全都不能用"，但去掉其中任何一条的原因，其余的照样会失败。

---

## 2. 证据链

### E1 — AOSP 上游提交（决定性）

| 字段 | 值 |
|---|---|
| Commit | `7c7b4faef83543b2a6d79a1aad52f8a1f2913e41` |
| 提交信息 | **Don't allow adb system app uninstall for user without root** |
| 作者 | Sanjana Sunil（Google，提交元数据见原始链接） |
| 日期 | 2026-01-21 |
| Bug | 296534248 / 422170772 |
| 修改文件 | `services/core/java/com/android/server/pm/PackageManagerShellCommand.java` |

补丁内容（`runUninstall()`）：

```diff
                 if (LocalServices.getService(UserManagerInternal.class)
                         .getUsers(UserManagerInternal.USER_FILTER_WITH_ALL_COMPLETE_USERS)
                         .size() > 1) {
+                    if (Binder.getCallingUid() != Process.ROOT_UID) {
+                        pw.println("Failure [only root can delete system app for a "
+                                + "particular user]");
+                        return 1;
+                    }
                     // If we are being asked to delete a system app for just one user, set flag so
                     // it disables rather than reverting to system version of the app.
                     flags |= PackageManager.DELETE_SYSTEM_APP;
```

> 第 1 节那句报错的**字面文本**就来自这里。这不是 OEM 措辞，是 AOSP 原文。

来源：<https://github.com/GrapheneOS/platform_frameworks_base/commit/7c7b4faef83543b2a6d79a1aad52f8a1f2913e41>
（该仓库是 AOSP 的镜像；此提交作者单位为 `google.com`，非 GrapheneOS 改动。）

### E2 — API 36 / API 37 的 A/B 对照

`trinadhthatakula/Thor` issue #316：

> "At the shell uid on Android 17, `pm uninstall -k --user N` on a system app is refused outright … **API 36 answers `Success` to the same command.**"

同一程序、同一身份、同一条命令，仅系统 API 级别不同，结果相反。

来源：<https://github.com/trinadhthatakula/Thor/issues/316>

### E3 — 跨项目独立复现

| 项目 | 报了什么 |
|---|---|
| `SmartPack/PackageManager` #172 | "**Android 17 change**: `adb shell pm uninstall -k --user 0` no longer works without root" |
| `UAD-NG` #1436 | "Uninstall doesn't work on **Android 17**" |
| `brevent/Brevent` #1772 | 同一报错，且 `support disable: false`（disable 路线也不可用） |

来源：[#172](https://github.com/SmartPack/PackageManager/issues/172) ·
[#1436](https://github.com/Universal-Debloater-Alliance/universal-android-debloater-next-generation/issues/1436) ·
[#1772](https://github.com/brevent/Brevent/issues/1772)

### E4 — 关键定位：拦截在"外壳"，不在"服务端"

上引补丁改的是 `PackageManagerShellCommand` —— **`pm` 命令的解析器/外壳**，不是 `PackageManagerService.deletePackageVersioned` 本体。

这解释了为什么"绕过外壳"能成功：直接以 Binder 调用服务端 API 时，这段检查根本不参与。

对照实验（同一台 Android 17 设备）：

```
$ adb shell pm uninstall --user 0 <系统应用>        → 平台拒绝（E1 的检查）
$ adb shell pm uninstall <系统应用>（不带 --user）  → DELETE_FAILED_INTERNAL_ERROR
```

第二条走的是"彻底删除系统应用"，需要 `DELETE_SYSTEM_APP` 权限，shell 没有，所以在**服务端**就被挡 —— 那是另一条路径，与 E1 无关。

### E5 — 已有的绕过实现

`Bingblop/ADB-Root-Toolbox` PR #38（v7.9.2 release notes）：

> 当 `pm uninstall --user 0` 被拒时，改为通过**直连 Binder 调用 `IPackageManager.deletePackageAsUser`**，以 `app_process` 形式在既有特权 shell（ADB 或 Shizuku）身份下运行。据称 App Manager 与 Canta 使用同一手法。

来源：<https://github.com/Bingblop/ADB-Root-Toolbox/pull/38>

### E6 — 本机实测数据

| 项 | 值 |
|---|---|
| 设备 | K100 PRO MAX |
| 构建 | `OS4.0.7.0.XGNCNXM` |
| Android | 17（`ro.build.version.sdk = 37`） |
| 补丁级别 | 2026-10-01 |
| root | 无（`su` 不存在，`ro.debuggable = 0`） |

**前置检查：shell 实际上拥有所需权限**

```bash
$ adb shell dumpsys package com.android.shell | grep "granted=true" \
    | grep -E "DELETE_PACKAGES|CHANGE_COMPONENT_ENABLED_STATE|SUSPEND_APPS"
      android.permission.CHANGE_COMPONENT_ENABLED_STATE: granted=true
      android.permission.SUSPEND_APPS: granted=true
      android.permission.DELETE_PACKAGES: granted=true
```

> 这一点很关键：**失败不是权限问题**。shell 有 `DELETE_PACKAGES`，却依然被拒——因为拦它的是 E1 的代码级检查，不是权限系统。

**`pm hide` 路线同时被堵**

```bash
$ adb shell pm hide --user 0 <pkg>
java.lang.SecurityException: Access denied, requires: android.permission.MANAGE_USERS
```

shell 未申请该权限，Android 17 的 AIDL 层 `@EnforcePermission` 直接拦住。

### E7 — 绕过后的结果与持久性

用 Shizuku（13.6.0）+ Canta（4.1.0）对同一系统应用执行卸载后：

```
$ adb shell pm list packages | grep <pkg>              → 无输出
$ adb shell pm list packages -u | grep <pkg>           → package:<pkg>   # 存在但"对用户卸载"
$ adb shell dumpsys package <pkg> | grep "User 0:"
    User 0: ... installed=false ... ceDataInode=-1 ... stopped=true ...
$ adb shell pm path <pkg>                              → 空
$ adb shell pidof <pkg>                                → 无
$ adb shell cmd package resolve-activity --brief <pkg> → No activity found
```

**重启后（开机 43 秒）复查，以上六项全部保持不变。**

`ceDataInode = -1` 是关键判据：表示数据目录被显式删除；而"出厂就对用户未安装"的包是 `ceDataInode = 0`。后者常见于厂商用自家组件替代的 AOSP 组件（例如用自家安装器替代 `com.android.packageinstaller`）。

---

## 3. 根因总结

```
Android 17 (API 37)
  └─ PackageManagerShellCommand.runUninstall()
        └─ 新增：caller != ROOT_UID 时直接拒绝"系统应用 + 指定用户"
              └─ 影响所有 OEM（Pixel / 三星 / 小米 / GrapheneOS …）

部分 OEM（如小米 HyperOS）
  └─ PackageManagerServiceImpl.shouldRestrictEnabledSettingsChange()
        └─ 在 setEnabledSettings 上额外拦截 shell 改启用状态
              └─ 使 pm disable-user / disable-until-used / 组件级 disable 同样失效
```

两条限制**独立存在**，叠加后即"非 root 无路可走"。

---

## 4. 为什么"直接进文件系统删 APK"也不行

系统分区不是权限问题，而是**不可变设计**：

```
/dev/block/dm-XX on /product type erofs (ro, ...)
ro.boot.veritymode    = enforcing
ro.boot.vbmeta.device_state = locked
```

- `erofs` 是只读文件系统格式，`rm` 返回 `EROFS`，**root 也改不动**
- 之上还有 dm-verity 逐块校验 + AVB 签名

所以 Android 上"删除系统应用"的正确等价物是 **overlay 遮蔽**（Magisk）或 **`pm uninstall`（root）**，而不是 `rm`。

---

## 5. 可行的非 root 绕过

原理：以 shell（uid 2000）身份运行一个进程，**绕开 `pm` 外壳**，直接 Binder 调用服务端 API。

现成工具链：

| 组件 | 作用 | 许可证 |
|---|---|---|
| [Shizuku](https://github.com/RikkaApps/Shizuku) | 提供 shell 身份的特权通道，无需 root | Apache-2.0 |
| [Canta](https://github.com/samolego/Canta) | 图形界面，执行直连 Binder 的卸载 | LGPL-3.0 |

### 5.1 启动 Shizuku（非 root）

> ⚠️ **`sh /sdcard/Android/data/moe.shizuku.privileged.api/start.sh` 在 13.6.0 上已不存在**（该文件不会被生成）。正确做法是直接执行 APK 内的 `libshizuku.so`：

```bash
APK=$(adb shell pm path moe.shizuku.privileged.api | tr -d '\r' | sed 's/package://')
LIBDIR=$(dirname "$APK")/lib/arm64          # 按设备 ABI 调整
adb shell "$LIBDIR/libshizuku.so"
```

输出示例：

```
info: starter begin
info: starting server...
info: shizuku_server pid is <pid>
info: shizuku_starter exit with 0
```

启动命令也可在 Shizuku App「通过连接电脑启动（使用 adb）→ 查看指令」中直接查看。

### 5.2 执行卸载

在 Canta 中授权并选择目标包卸载。Shizuku 侧会弹出授权提示，允许后应用列表才会正常显示。

### 5.3 恢复

```bash
adb shell pm install-existing --user 0 <pkg>
```

系统 APK 从未被删除，可随时恢复。

---

## 6. 适用范围与已知限制

| 项 | 说明 |
|---|---|
| 样本数 | **仅 1 台设备实测**，不代表所有 OEM 行为一致 |
| 只影响"系统应用 + 指定用户" | `pm uninstall --user 0 <普通应用>` 在任何 Android 版本仍可用 |
| 部分 OEM 可能在服务端也拦 | 若如此，Shizuku 方案同样无效，只剩 root |
| 恢复出厂设置 | 会还原被卸载的包 |
| 系统大版本 OTA 后 | 建议复查：`pm list packages \| grep <pkg>` |
| 副作用 | 被卸载组件的功能入口会表现为"点击无反应"，而非友好报错 |
| Shizuku 无 root 时 | 每次重启需重新启动（可用无线调试免电脑） |

---

## 7. 快速自检

判断你遇到的是不是同一个问题：

```bash
# 1) 系统版本是否 API 37+
adb shell getprop ro.build.version.sdk          # 37 = Android 17

# 2) 目标是否为系统应用
adb shell pm list packages -s | grep <pkg>

# 3) 是否卡在 E1 的检查（报错文本匹配）
adb shell pm uninstall --user 0 <pkg>           # 期望：Failure [only root can delete ...]

# 4) 若 disable 也失败，判断是否为 OEM 定制
adb shell pm disable-user --user 0 <pkg> 2>&1 | head -3
```

---

## 8. 参考

- AOSP 提交（AOSP 镜像）：<https://github.com/GrapheneOS/platform_frameworks_base/commit/7c7b4faef83543b2a6d79a1aad52f8a1f2913e41>
- Thor #316（API 36/37 对照）：<https://github.com/trinadhthatakula/Thor/issues/316>
- SmartPack/PackageManager #172（Android 17 变更报告）：<https://github.com/SmartPack/PackageManager/issues/172>
- UAD-NG #1436（同问题复现）：<https://github.com/Universal-Debloater-Alliance/universal-android-debloater-next-generation/issues/1436>
- Brevent #1772（OEM 侧 disable 失效）：<https://github.com/brevent/Brevent/issues/1772>
- ADB-Root-Toolbox PR #38（Binder 直调实现说明）：<https://github.com/Bingblop/ADB-Root-Toolbox/pull/38>
- Shizuku：<https://github.com/RikkaApps/Shizuku>
- Canta：<https://github.com/samolego/Canta>

---

## 免责声明

**内容生成方式**：本文由 AI 助手起草。实验在真实设备上执行，数据为实测记录，但与实验无关的分析、推论与措辞均由 AI 完成。**AI 生成部分可能出错，作者不为该部分内容的准确性作保证**；如与官方文档、源码或原始 issue 不符，请以原始来源为准。

**需要自行判断的地方**：文中所有"根因""为什么"类解释，都是在观察数据之上做的推断，不是任何人写下的定论。请把它当作**待验证的假设**，而不是结论。

**技术声明**：本文记录的是平台行为的技术分析，不构成对任何设备的操作建议。卸载系统组件可能影响设备功能、保修与后续 OTA 行为；请在理解风险的前提下自行判断，并保留恢复手段（`pm install-existing`）。
