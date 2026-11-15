# 容器注入 Hachimi：进展与卡点（2026-11-16 凌晨）

> 承接 `docs/CONTAINER-PROGRESS-2026-11-11.md`。
> 结论先行：**Hachimi 已在容器里完整加载并 hook 成功（国服区域识别正确），
> 但 `Hooking finished` 后数秒内进程 SIGBUS 退出——卡在 Hachimi 的 inline hook 与容器环境的兼容性。**

---

## 一、已打通的链路（全部实测）

| 环节 | 做法 | 证据 |
|---|---|---|
| 容器跑 Android 16 | `ALEX5402/NewBlackbox` 自编译（JDK21 + NDK 29） | 能启动、能进列表 |
| 容器内 WebView | `WebViewProxy` 把 guest Context 换成宿主 Context | `ERR_CACHE_MISS` 归零，协议页通过 |
| 读 `/sdcard` | manifest 必须声明 `MANAGE_EXTERNAL_STORAGE`（只给 appops 不够） | `Permission denied` 消失 |
| 原版游戏在容器里跑 | 不改任何字节，guest 从系统已装包导入 | 到 bilibili 登录页 |
| **Hachimi 注入** | **`nativelib` 目录重定向**：`libmain.so`→Hachimi，`libmain_orig.so`→原版 | 见下 |

### 关键机制（这两条是今晚最值钱的发现）

1. **Hachimi 的 `JNI_OnLoad` 必须能 `dlopen("libmain_orig.so")`**
   源码 `src/android/main.rs`：
   ```rust
   const LIBRARY_NAME: &CStr = c"libmain_orig.so";
   let handle = libc::dlopen(LIBRARY_NAME.as_ptr(), RTLD_LAZY);
   orig_fn = dlsym(handle, "JNI_OnLoad");   // 原版 Unity 入口
   if !Hachimi::init() { return orig_fn(vm, reserved); }
   hook::init(env.get_raw());
   orig_fn(vm, reserved)
   ```
   缺这个软链时，Hachimi 拿到空指针 → 反复重入 → 栈溢出（512 帧同地址 SIGBUS）。

2. **guest 的 `nativeLibraryDir` 必须被改写**（在 `createPackageContext(applicationInfo)` 之前）
   否则 guest 从系统里那个**真实游戏 lib 目录**加载原版 `libmain.so`，我们的替换不生效。
   做法：建 `/data/user/0/top.niunaijun.blackbox/files/nativelib/`，
   其它 `.so` 全部软链到游戏真实 lib 目录，只有：
   ```
   nativelib/libmain.so      -> files/libmain.so        （Hachimi 的 so）
   nativelib/libmain_orig.so -> 游戏原版 lib/arm64/libmain.so
   ```
   实测日志：`BB_INJECT: redirect -> .../nativelib libmain=15068464 origMain=true`

### 成功日志（新 Hachimi 版）

```
BB_INJECT: guest processName=com.bilibili.umamusu
BB_INJECT: redirect -> /data/user/0/.../nativelib libmain=15155040 origMain=true
Hachimi: hachimi::core::hachimi: Game region: China
Hachimi: hachimi::android::main: JNI_OnLoad
Hachimi: hachimi::il2cpp::hook::umamusume::Device: new_hook!: IsIllegalUser
...
Hachimi: hachimi::il2cpp::hook: Hooking finished
```

---

## 二、当前卡点：SIGBUS

- `dumpsys activity exit-info top.niunaijun.blackbox` → `process=top.niunaijun.blackbox:p0 reason=2 (SIGNALED) status=7`
- status=7 = **SIGBUS**
- 已排除：
  - 不是 GUI：配置 `disable_gui:true` 后仍然崩
  - 不是版本：`v0.14.0-5ea0359`（15,068,464 B）与 `v0.14.0-2deadd3`（15,155,040 B）都崩
  - 不是区域识别：日志里 `Game region: China` 正确
- 最可能原因：Hachimi 的 inline hook（自定义 trampoline / mprotect 代码页）与 BlackBox 的内存映射或它自己的 API hook 冲突，第一次调用被 hook 的函数就触发 BUS_ADRERR

## 三、Hachimi 配置（已生效，可复用）

路径：`/sdcard/Android/media/com.bilibili.umamusu/hachimi/config.json`

```json
{"language":"zh-cn","disable_gui":true,"disable_gui_once":false,"debug_mode":true,"target_fps":-1}
```
注意：`language` 只接受 `en` / `zh-tw` / `zh-cn` / `vi`（大小写敏感，写 `zh-CN` 会 `Init failed`）。

## 四、下一步（按成本排）

1. **从源码编 Hachimi，逐个关掉 hook**（`src/il2cpp/hook/umamusume/*.rs`），先保留最小集合（Time/fps 相关），
   确认哪个 hook 一调用就 BUS —— 这是唯一能定位的办法。
2. **给 Hachimi 的 interceptor 加日志**：在被 hook 函数入口打印，最后一条就是崩点。
3. 如果最终确认是 BlackBox 的内存布局冲突：换一个容器分支（`NPC2000/NewBlackbox-pro`）再试。

## 五、产物与脚本位置

- 容器构建：`.github/workflows/blackbox-build.yml`（固定签名 `ci/bb.jks`，可 `pm install -r` 覆盖升级，容器数据不丢）
- `nativelib` 重定向代码在 `BActivityThread.java` 的 `createPackageContext(applicationInfo)` 之前注入
- Hachimi so：`/sdcard/Download/libmain.so`（容器启动时自动复制到 `files/`）
- 日志：`logcat -d | grep -E "BB_INJECT|hachimi"`