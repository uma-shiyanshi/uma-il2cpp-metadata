# 决定性发现：崩溃来自 Hachimi 的 linker `do_dlopen` hook（2026-11-16 04:10）

承接 `docs/HOOK-BISECT-PLAN-2026-11-16.md`。

## 一、配置终于生效了（关键前提已解决）

配置**必须由 guest 进程自己写**，宿主进程写的不算：

```
BB_INJECT: cfg srcReadable=true dirWritable=true dstExists=true
BB_INJECT: cfg written len=1262 canReadBack=true
→ 日志出现大量 [DISABLED] new_hook!: ...   （65 个 hook 全部禁用生效）
```

原因：宿主进程（App.onCreate）写 `/sdcard/Android/media/...` 时，写入方上下文与 guest 不一致，
Hachimi 里 `fs::metadata(config_path)` 失败后会**静默回退默认配置**。
把写配置的代码放到 `BActivityThread`（运行在 guest 进程内）之后，立刻生效。

## 二、禁用全部 il2cpp hook + 关 GUI 后，仍然崩

| 配置 | 结果 |
|---|---|
| 全部 65 个 hook 禁用 + `disable_gui:true` | 仍然崩溃，信号从 **SIGBUS(7)** 变成 **SIGSEGV(11)** |
| 崩溃时间点 | `Hooking finished` 之后约 4 秒 |

结论：**问题不在游戏相关 hook，而在 Hachimi 的原生初始化**。

## 三、源码定位：`src/android/hook.rs::init_internal`

```rust
fn init_internal(env) -> Result<(), Error> {
    let hachimi = Hachimi::instance();
    let force_hook_dlopen = config.android.hook_libc_dlopen || /vendor/waydroid.prop 存在;

    // ① 无条件 hook linker 的 do_dlopen（或 dlopen）
    let dlopen_orig = Interceptor::find_symbol_by_name(LINKER_MODULE, DO_DLOPEN_V26)?;
    hachimi.interceptor.hook(dlopen_orig, dlopen_hook)?;

    // ② 只有 GUI 开启时才 hook RegisterNatives
    if !config.disable_gui { ... hook(RegisterNatives) ... }
}
```

`disable_gui:true` 后**只剩 ①**：对 `linker64` 里 `__dl__Z9do_dlopenPKciPK17android_dlextinfoPKv` 的 inline hook。

在 BlackBox 里，linker 是所有 guest 共享的，`do_dlopen` 被改写后，**游戏后续每一次加载 so 都可能走坏**（游戏启动阶段会连续 dlopen `libtp2.so`、`libcri_*` 等）→ SIGSEGV。

这与观察完全吻合：崩溃点总是在 `Hooking finished` 之后 4 秒左右，也就是游戏开始批量加载原生库的时刻。

## 四、下一步：从源码编译，跳过 dlopen hook

### 补丁（`src/android/hook.rs`）
```rust
// 新增配置字段（src/core/hachimi.rs Config）
#[serde(default)]
pub disable_dlopen_hook: bool,

// init_internal 里
if !hachimi.config.load().disable_dlopen_hook {
    info!("Hooking {} at {:#x}", dlopen_name, dlopen_orig);
    hachimi.interceptor.hook(dlopen_orig, dlopen_hook)?;
}
```

### 同时建议改 `src/android/game_impl.rs`
```rust
pub fn get_data_dir(_package_name: &str) -> PathBuf {
    PathBuf::from("/data/local/tmp/hachimi")   // 容器内一定可读写
}
```

### CI 构建（新增 `hachimi-build.yml`）
```bash
rustup target add aarch64-linux-android
export CC_aarch64_linux_android=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android29-clang
export AR_aarch64_linux_android=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-ar
cargo build --release --target aarch64-linux-android
# 产物 target/aarch64-linux-android/release/libhachimi.so → 替换 /sdcard/Download/libmain.so
```

## 五、拿到新 so 后的验证顺序

1. `disable_dlopen_hook: true` + 全部 hook 禁用 → 期望：游戏正常跑到标题（Hachimi 只加载不动）
2. 放开 GUI（`disable_gui:false`）→ 期望：出现 Hachimi 悬浮菜单
3. 逐组放开 hook（先 `GameSettings`/fps 相关，再 `Story*`）→ 找出真正会崩的那一组