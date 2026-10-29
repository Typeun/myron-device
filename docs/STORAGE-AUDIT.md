# myron 存储边界审计（2026-10-29）

`board.toml` 的 `[userdata]` 假设 LUN 0 上有一个 128 GiB 的 `unixnu` GPT 分区。
今天在连着 adb 的这台机器上核对，结论是：**这个分区不存在，写进去会覆盖 Android 的 userdata。**

## 实测

```text
$ adb shell 'su -c "ls /dev/block/by-name"'
userdata                     # 没有 unixnu

$ adb shell 'su -c "cat /sys/class/block/sda34/start /sys/class/block/sda34/size"'
30380032                     # 512 字节扇区
998756312
```

换算成 4096 字节 LBA：

| 项 | 值 |
| --- | --- |
| userdata 起 | `30380032 / 8` = `0x39f200` |
| userdata 止 | `(30380032 + 998756312) / 8 - 1` = `0x7aaebfa` |
| 大小 | 476.24 GiB |

`board.toml` 里 `unixnu` 的窗口是 `0x5aaeb00..0x7aaeaff`，**完全落在上面这段 live userdata 内**。
只要 `native-ufs-root` 真的按这个窗口写入，就是几百 GiB Android 数据的覆盖。

另外注意 `board.toml` 旧注释里的 HyperOS userdata（`0x39f200..0x5aaeaff`）也与实测不符：
现在 userdata 一直长到 `0x7aaebfa`，说明 10-21 那次 GptSplitDxe 切分没有留在这台机器上
（可能被重刷/恢复出厂撤销了）。

## 两条可选路线

### A. 保持双系统：先切分（有数据风险，必须先备份）

1. 备份 `/data`（或至少 DCIM/文档）；切分要缩 userdata，等于重建文件系统。
2. 用 myronPkg 里的 `GptSplitDxe` 在 UEFI 里切出 `unixnu`，或把 `userdata` 缩到 `0x5aaeaff`。
3. 切完重新核对：

   ```bash
   adb shell 'su -c "ls /dev/block/by-name | grep -E \"unixnu|userdata\""'
   adb shell 'su -c "cat /sys/class/block/sda34/start /sys/class/block/sda34/size"'
   ```

   只有当第二行仍是 `30380032 998756312` 且多出 `unixnu` 时，`first_lba`/`last_lba` 才可照抄。
4. 把切分后的实际 LBA 填回 `board.toml`，再继续 `unixnu build`。

### B. 单系统：直接用 userdata（会清掉 Android 数据）

与同板子的 `songyuan-device` 一样，把 `[userdata]` 改成：

```toml
[userdata]
partition = "userdata"
first_lba = 0x39f200
last_lba = 0x7aaebfa
lba_bytes = 4096
```

这条路线不需要动 GPT，`fastboot boot` 后 iOS 直接占用 userdata；Android 数据没了，
重新刷 HyperOS 才能回去。**只在已经决定不要这台机器 Android 数据时使用。**

## 为什么没有替你改

`board.toml` 的写目标是块设备。改 `partition` 或 LBA 等于替你决定"要不要保住 Android 数据"，
这件事不能由工具默认。上面的注释已经写进 `board.toml`，两条路线的值都在，选一条执行即可。

## 复核命令

```bash
# 每轮 fastboot boot 前都跑一次
adb shell 'su -c "cat /sys/class/block/sda34/start /sys/class/block/sda34/size"'
python - <<'PY'
start, size = map(int, input().split())
print(hex(start // 8), hex((start + size) // 8 - 1))
PY
```
