# 到 Mac 之前 / 之后的边界

本文记录 2026-10-29 在 Windows 上实际完成的部分，以及必须换到 macOS（或 Linux）才能继续的那一步。
所有命令都从 **`unixnu-myron`**（fork）仓库执行，设备仓库是 `../myron-device`。

## 已经在 Windows 上做完并验证

| 步骤 | 结果 | 证据 |
| --- | --- | --- |
| 设备仓库解析 | 15 个 driver、7 个 kit、device package 全部解析到 `out/myron` | `unixnu status` |
| 构建树组装 | `out/myron/tree/`，模块用 junction 链接（Windows 无 symlink 权限时的回退） | `unixnu tree`，exit 0 |
| 子模块 | 21 个 submodule 全部在 fork 的 pin 上初始化 | `git submodule status` |
| ADT | `out/adt/myron.adt`，13,476 字节，8 核、GIC、iPhone15,2 身份、root UUID 正确 | `tools/dump-adt-tree.py` |
| 固件 | `SILICIUM_UEFI.fd` 3,145,728 字节，sha256 `d7a3d2f3…` | GitHub Actions run 40821624925 |
| 内核缓存 pin | 解压后 arm64 切片 sha256 `f18b4df8…`，与 `board.toml` 的 `kernelcache_sha256` **一致** | `tools/decode-kernelcache.py` |
| trust cache | `7416ca4f…`，与 `board.toml` 的 `trust_cache_sha256[0]` **一致** | sha256sum |

## 必须 Mac（或 Linux）的一步

**编译 XNU。** 这一步需要 Xcode 的 iPhoneOS SDK 与 Apple 的 `compression_tool`，Windows 没有替代品。
从这里开始的所有产物（`kernel.macho`、package 的 archive、`payload.img`、`boot.img`）都依赖它。

```bash
# 0. 依赖：Xcode（含 iPhoneOS SDK）、brew 安装 llvm 与 mtools
xcodebuild -version
brew install llvm mtools

# 1. 设备仓库与源码
cd unixnu-myron
./unixnu defconfig myron_defconfig -D ../myron-device
./unixnu fetch -D ../myron-device
./unixnu tree  -D ../myron-device

# 2. 内核（Mac）
./unixnu build kernel -D ../myron-device
#    产出 out/myron/kernel/kernel.macho

# 3. 设备包与 archive
./unixnu build package -D ../myron-device

# 4. payload：内核 + ADT + trust cache；--unique-chip-id 是必需的平台种子
./unixnu build payload -D ../myron-device \
  --trust-cache ../myron-device/trustcache.bundle \
  --unique-chip-id <你的平台种子>

# 5. 固件与 boot image
./unixnu build firmware -D ../myron-device --uefi ../myron-uefi
./unixnu bootimg -D ../myron-device --uefi ../myron-uefi \
  --dtb ../myron-device/canoe.dtb
#    payload.img 已存在时会自动当作 ramdisk；--dtb 也可用 UNIXNU_DTB

# 6. 真机
fastboot boot out/myron/boot/boot.img
```

## 还没准备好的输入

1. **`--unique-chip-id`**：`board.toml` 写的是 `"required"`，仓库里没有这个种子（也不该有）。
   本机生成 ADT 时用的是占位值 `0x1`，**真机 boot 前必须换成你自己的值**，否则 IOPlatformUUID 每次不同。
2. **`--data-ramdisk`**：Data 种子（`scripts/seed-data-*` 的产物）。没有它也能进锁屏，但没有持久 Data。
3. **`unixnu` 分区**：还不存在，见 [STORAGE-AUDIT.md](STORAGE-AUDIT.md)。不解决就只能用 RAM Data，
   或者走单系统写 `userdata`。

## 把这些带回给我

Mac 上跑完后把这几样发回来，我接着做真机侧：

- `out/myron/kernel/kernel.macho` 的大小与 sha256
- `out/myron/payload/payload.json`（有效启动参数）
- `out/myron/boot/boot.img` 的大小与 sha256
- 每一步的 `out/myron/logs/*.log` 最后 50 行
