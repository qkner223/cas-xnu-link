# 在 GitHub 上把 cas 的 XNU 内核链出来

这台 Linux 机器上编译已经 **0 失败**，唯一过不去的是**链接**：`ld64.lld` 不认
`-image_base` / `-segaddr`，链出来的内核基址是 `0x0` 而不是 `0xfffffff007004000`，跳进去必炸。

GitHub Actions 的 **`macos-14` runner 是 arm64 + 完整 Xcode**，上面就有真正的 Apple `ld64`。
所以把「链接所需的最小集」打成一个包，让 runner 链一下就完事——不需要源码树、不需要 iOS SDK、
不需要你本地有 Mac。

## 包里有什么（49.9 MB）

```
gh-link-kit.tar.gz
├── objs/     1046 个原 macOS 构建出来的 .o（顺序来自 meta/link.filelist）+ 4 个 root 对象
├── install/    14 个本次改过的 .o（hk_battery / hk_wifi / umi_* / HKBluetooth …）
│               —— 在 Linux 上用 clang 21、完整 XNU CFLAGS（含 -mkernel）编出来的 Mach-O arm64
├── libs/      libfirehose_kernel.a、libpthread_kext.a
├── meta/      .CFLAGS、.LDFLAGS、all-alias.exp、symbolsets.plist、link.filelist
└── link.sh    自动重写路径、替换对象、补空 libcc_kext、链接、验收
```

链接参数原样保留，包括这几条关键项：

```
-Wl,-image_base,0xfffffff007004000
-Wl,-segaddr,__PRELINK_TEXT,0xfffffff004004000
-Wl,-alias_list,meta/all-alias.exp
-Wl,-sectcreate,__LINKINFO,__symbolsets,meta/symbolsets.plist
```

## 三步

1. **建仓库**（私有即可），把这两个文件传上去：
   ```
   gh-link-kit.tar.gz
   .github/workflows/link.yml
   ```
   网页拖放也行，命令行也行：
   ```bash
   cd gh-link
   git init -b main && git add -A && git commit -m "link kit"
   git remote add origin git@github.com:<你>/<仓库>.git
   git push -u origin main
   ```

2. **跑 workflow**：仓库 → Actions → `link-xnu-kernel` → Run workflow。
   macos-14 上大约 1–3 分钟。

3. **下载 artifact**：这次运行页面底部 `mach.kernel` → 下载 → 解压 → 把文件路径给我。

## 我这边接着做什么

```bash
# 1) 校验
llvm-nm mach.kernel | grep " __start$"     # 必须是 fffffff0074a14a8
# 2) 打进 payload
python3 scripts/repack_payload.py out/payload.bin payload-v4.bin 1800 --kernel mach.kernel --ramdisk payloads/root.apfs
# 3) 刷
PAYLOAD=payload-v4.bin LOGWAIT=150 ./cas-ios-flash/flash.sh boot
```

预期变化（对照这次的日志）：

- `BT-cas:` 之后会出现 `HKBluetooth: device-tree node "bluetooth" published, local-mac-address 02:52:44:57:00:02`
- `bluetoothd` 不再 `exit(0)`，内核计数 `H4 client` 从 **0** 变成 ≥1
- 电池摘要里 `V=` 不再是 `-1`（golden 的 `hk_battery.c` 这次一起链进去）

## 验收标准

| 项 | 期望 |
|---|---|
| 产物大小 | ~20 MB（这次是 20,016,112） |
| `__start` | `fffffff0074a14a8` |
| 段基址 | `__TEXT 0xfffffff007004000`、`__PRELINK_TEXT 0xfffffff004004000` |
| 新增字符串 | `device-tree node "bluetooth" published` |

如果 `__start` 不是那个地址，说明 `-image_base` 没生效——把 `link.sh` 的输出贴给我，我看参数。
