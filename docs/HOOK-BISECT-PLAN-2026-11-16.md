# Hook 二分定位：进展与下一步（2026-11-16 04:00）

承接 `docs/CONTAINER-INJECT-2026-11-16.md`。

## 一、这轮新增的确定结论

| 结论 | 证据 |
|---|---|
| 崩溃是 **SIGBUS**（不是 ACE kill） | `dumpsys activity exit-info` → `p0 reason=2 (SIGNALED) status=7` |
| 与 GUI 无关 | `disable_gui` 后仍崩（早期配置可读时验证过） |
| 与 Hachimi 版本无关 | `v0.14.0-5ea0359` 与 `v0.14.0-2deadd3` 都崩 |
| **`new_hook!` 有 0 地址保护**，不是空指针 hook | 源码 `src/il2cpp/hook/mod.rs`：`if ($orig != 0) { ... } else { error!("{} is null") }` |
| **支持 `disabled_hooks` 配置**，可逐组禁用 hook | 源码 `src/core/hachimi.rs:302` `disabled_hooks: FnvHashSet<String>`；`hook/mod.rs:7` 判断 |
| Hachimi 全部 hook 名共 **65 个** | 从源码 `new_hook!(addr, Name)` 提取，见下 |

## 二、卡点：容器里的 Hachimi 读不到 `/sdcard` 上的配置

- 配置路径（源码 `src/android/game_impl.rs`）：`/sdcard/Android/media/<package>/hachimi/config.json`
- 实测：文件内容正确、位于正确路径，但 **Hachimi 日志里 0 条 `[DISABLED]`**（配置未生效）
- `load_config` 用 `fs::metadata(&config_path).is_ok()`，失败时**静默回退默认值**，不会报错
- 已给 uid 10805 单独开 `MANAGE_EXTERNAL_STORAGE` / `LEGACY_STORAGE`（`cmd appops set --uid 10805 ...`），仍无效
- 推测：guest 进程里 Rust 的 `stat()` 走的路径与 Java 侧不同（BlackBox 的存储重定向或 SELinux 上下文差异）

## 三、自动化已具备的能力（可无人值守）

容器内 `App.onCreate`（主进程）在检测到私有目录标记时，自动执行：

```
files/dsh-auto-launch 存在 → 写配置 → 未安装则 installPackageAsUser → launchApk
```

- 标记创建：`run-as top.niunaijun.blackbox sh -c 'touch files/dsh-auto-launch'`
- 配置来源：`/sdcard/Download/hachimi_config.json` → 复制到 media 路径
- 容器更新：固定签名 `ci/bb.jks`，`pm install -r` 覆盖，数据不丢
- 产物：release `builds` 里的 `bb-cfg-*` / `bb-auto2-*` / `bb-arm64-*`

## 四、下一步（两条路，都无需用户点屏）

### 路线 1：重编 Hachimi，把 data_dir 改到可写位置（最稳）
```rust
// src/android/game_impl.rs
pub fn get_data_dir(_package_name: &str) -> PathBuf {
    PathBuf::from("/data/local/tmp/hachimi")   // 或 /data/user/0/top.niunaijun.blackbox/files/hachimi
}
```
然后 CI：`rustup target add aarch64-linux-android` + NDK clang 作为 linker，`cargo build --release --target aarch64-linux-android`。
拿到 so 后按原流程替换 `/sdcard/Download/libmain.so`，即可用 `disabled_hooks` 做二分。

### 路线 2：验证 guest 的 /sdcard 视图
在容器里加一行代码：guest 进程 `new java.io.File("/sdcard/Android/media/com.bilibili.umamusu/hachimi/config.json").canRead()`，
把结果打到 `BB_INJECT`，就能确认是"看不到"还是"看得到但 Rust 读不到"。

## 五、二分策略（拿到能用的配置后）

1. `disabled_hooks` = 全部 65 个 → 若游戏稳定，说明崩在 hook 层
2. 二分：一半一半，每次改 `/sdcard/Download/hachimi_config.json` → 重启容器主进程 → 自动导入/启动 → 看 exit-info `status`
3. 收敛到单个 hook 名后，到源码里看它的 `get_method_addr` 与参数数是否与国服匹配