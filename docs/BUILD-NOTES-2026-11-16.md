# Hachimi 源码构建笔记（2026-11-16 04:20）

配套文档：`docs/DLOPEN-HOOK-ROOTCAUSE-2026-11-16.md`（为什么需要自编译）

## 一、已跑通的 CI 流水线

`.github/workflows/hachimi-build.yml`，步骤：

1. 安装 rustup + `aarch64-linux-android` target
2. clone `Hachimi-Hachimi/Hachimi-Unity2020` 的 `2020` 分支
3. 打两个补丁：
   - `src/core/hachimi.rs` 的 `Config` 增加 `#[serde(default)] pub disable_dlopen_hook: bool,`
   - `src/android/hook.rs` 的 `init_internal`：把 `hachimi.interceptor.hook(dlopen_orig, ...)` 包进
     `if !config.disable_dlopen_hook { ... }`
4. `cargo build --release --target aarch64-linux-android`
5. 产物上传到 release `builds`，文件名 `libmain-nodlopen-*.so`

首次构建**成功**（14,816,648 B），但产物装进容器后 **dlopen 失败**：

```
dlopen failed: cannot locate symbol "_Znwm" referenced by libmain.so
```

`_Znwm` = C++ `operator new(unsigned long)`，说明二进制里有 C++ 目标文件，但没链 C++ 运行库。

## 二、尚未解决的构建问题

第二次构建（加 `CXXSTDLIB=c++_static`、`-static-libstdc++`）产物与第一次**字节完全相同**
（sha256 前 16 位都是 `72b2fe78554d`），说明这些环境变量没有生效——Rust 的 cdylib 链接阶段没吃到。

### 下次直接试这几种（按优先级）

1. 显式加链接参数：
   ```bash
   export RUSTFLAGS="-C link-arg=-lc++_static -C link-arg=-lc++abi"
   ```
2. 或改成动态 C++ 并在容器里补齐运行库：
   ```bash
   export RUSTFLAGS="-C link-arg=-lc++_shared"
   # 把 NDK 的 libc++_shared.so 一并放进 nativelib 目录（容器里已有软链机制）
   ```
3. 若符号来自某个 build.rs 编译的 C++ 静态库，用 `CXXSTDLIB=c++_static` 作用于该 crate，
   需要确认是哪个 crate（`cargo tree | grep -i cc`）。

### 自检命令（构建后必跑）
```bash
"$TC/llvm-readelf" -d libhachimi.so | grep NEEDED      # 期望没有 libc++_shared 且能自洽
"$TC/llvm-readelf" -s libhachimi.so | grep -w _Znwm    # 期望是「已定义」而不是 UND
```

## 三、装进容器后的验证顺序（产物能 dlopen 之后）

```bash
# 1. 换 so（容器下次启动会自动拷贝，按大小判断）
cp libmain-nodlopen.so /sdcard/Download/libmain.so

# 2. 配置：关 dlopen hook + 关 GUI + 禁全部 hook
cat > /sdcard/Download/hachimi_config.json <<'EOF'
{"language":"zh-cn","disable_gui":true,"disable_dlopen_hook":true,"debug_mode":true,"disabled_hooks":[...65 个...]}
EOF

# 3. 触发（标记存在时容器自动导入+启动）
run-as top.niunaijun.blackbox sh -c 'touch files/dsh-auto-launch'
am force-stop top.niunaijun.blackbox
monkey -p top.niunaijun.blackbox -c android.intent.category.LAUNCHER 1

# 4. 看结果
logcat -d -s Hachimi:I BB_INJECT:I BB_AUTO:I | grep -E "dlopen hook disabled|DISABLED|Hooking finished"
dumpsys activity exit-info top.niunaijun.blackbox | grep -E "reason=|process=top.niunaijun.blackbox:p0"
```

期望：`dlopen hook disabled by config` 出现、游戏不再 SIGSEGV、能到标题页。
之后再逐组放开 hook（先 fps/画面，再剧情类）。

## 四、顺带确认的事实

- 容器里 guest 的 uid 与宿主 App 相同（`u0_a805` / 10805），所以**guest 进程自己写**的
  `/sdcard/Android/media/com.bilibili.umamusu/hachimi/config.json` 是可读的；
  宿主进程（App.onCreate）写的那份 Hachimi 读不到（会静默回退默认配置）。
- 配置生效的直接证据是日志里出现 `[DISABLED] new_hook!: xxx`。