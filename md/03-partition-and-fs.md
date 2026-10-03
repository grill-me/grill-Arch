# 分区、文件系统、簇/块大小决策

针对机器：**16G 内存 + 240G NVMe**。

---

## 最终方案

```
nvme0n1 (240G)
├── p1  512M   ESP    → FAT32, 簇 32K（默认）
├── p2  4G     Swap   → Linux swap
└── p3  ~235G  Root   → ext4, 块 4K（默认）
```

---

## ESP 分区是什么

ESP = **EFI System Partition**，UEFI 规范要求的引导分区。

| 时代 | 引导分区 | 作用 |
|---|---|---|
| 老 BIOS | MBR（512 字节） | 告诉 CPU 去哪个扇区找 bootloader |
| 现代 UEFI | **ESP**（FAT 分区） | 放 bootloader 二进制、内核、initrd、EFI 配置 |

UEFI 固件开机第一件事：扫描磁盘找 ESP，从 ESP 里读 `EFI/BOOT/BOOTX64.EFI` 启动。

ESP 里装什么：

```
ESP/
├── EFI/
│   ├── BOOT/BOOTX64.EFI       ← 通用 fallback 启动器
│   └── arch/
│       ├── systemd-bootx64.efi
│       ├── shimx64.efi        ← Secure Boot 用
│       └── grubx64.efi
├── loader/                    ← systemd-boot 配置
├── vmlinuz-linux              ← 内核
└── initrd.img-linux           ← initramfs
```

一套内核 + initrd ≈ 100MB。**512M 够装 5 套**。

---

## 文件系统选择

### ESP 必须 FAT32（规范强制）

| 文件系统 | ESP 能用？ | 原因 |
|---|---|---|
| **FAT32** ✅ | 能 | UEFI 规范强制要求 |
| FAT16 | 能（ESP < 2G） | 已淘汰 |
| **NTFS** ❌ | 不能 | 规范不允许 |
| ext4 | ❌ | 规范不允许 |

**原理**：UEFI 固件内置 FAT 驱动，**没有** NTFS/ext4 驱动。固件开机只能读 FAT 格式。

### 根分区选 ext4（默认推荐）

| 文件系统 | 选它的理由 | 别选它的理由 |
|---|---|---|
| **ext4** ✅ | 最成熟、所有工具认、性能够、装错最多 | 无快照、无压缩、无校验和 |
| **btrfs** | 快照、内置压缩、校验和 | 比 ext4 复杂，排错门槛高 |
| xfs | 大文件、超大磁盘 | NVMe 上优势不明显 |
| F2FS | 纯 SSD，为闪存设计 | 现代 NVMe 控制器已处理大部分 GC，收益被抹平 |

**判断**：开发机/日常用 → **ext4**。想要系统快照（崩了回滚）→ btrfs。

> btrfs 装完要在 `/etc/mkinitcpio.conf` 的 HOOKS 里加 `btrfs`。

---

## 簇/块大小

### ESP（FAT32）：32K

| 簇大小 | ESP 512M 后果 | 建议 |
|---|---|---|
| 16K | 落在"标准簇表"边缘，可行但不必要 | ⚠️ |
| **32K** ✅ | 512M/32K = 16K 簇，Rufus 默认 | ✅ |
| 64K | 浪费略大，对 512M 无收益 | ❌ |

Rufus 默认就是 32K，别动。

### 根分区（ext4）：4K（默认，别加 -b）

| 块大小 | 内部碎片 | 大文件 IO | 你这机器值不值得 |
|---|---|---|---|
| **4K** ✅ | 极低 | 基准 | ✅ 免费最优 |
| 16K | 4× 于 4K | +1-2% | ❌ |
| 32K | 8× 于 4K | +2-3% | ❌ |
| 64K | 16× 于 4K | +3-5% | ❌（只有视频剪辑/数据库值得） |

**为什么 4K 是对的**：

- 现代 NVMe SSD 页大小 = 4KB
- 文件系统块对齐 SSD 页，IO 不拆包，延迟最低
- 4KB 是 SSD、文件系统、OS（PAGE_SIZE）三方公约数
- 32K/64K 要 SSD 控制器做"读写放大"，反而慢

**典型开发者内容分布**（240G 估算）：

| 内容 | 大小 |
|---|---|
| 系统 + 桌面 | ~8G |
| Node/Python/Java 工具链 | ~5G |
| 代码仓库（git 对象 1-4KB 小块） | ~5-10G |
| Docker/容器层 | ~10-20G |
| 文档/配置 | ~5G |
| 留空 | ~170G |

**git 对象 + 源码 + 依赖全是 KB 级小块**，任何 >4K 的块都是给"大文件 IO"优化的——你的大文件根本不在根分区。

**反例**：一个 10K 文件的代码仓库，200 个小文件：

| 块大小 | 浪费 |
|---|---|
| 4K | <1MB |
| 32K | ~6MB |
| 64K | ~12MB |

不致命，但没理由付这个代价。

---

## Swap 大小（16G 内存）

| 内存 | 推荐 swap |
|---|---|
| 8G | 8G |
| **16G** | **4G** |
| 32G+ | 2-4G 或 hibernate 文件大小 |

16G 用 4G swap 足够。**省下 4G 给根分区用**（240G 本来就不富裕）。

---

## 具体命令速查

```bash
# ESP（512M，FAT32，32K 簇）
mkfs.fat -F32 -C /dev/nvme0n1p1
# 或显式：
mkfs.fat -F32 -s 32 /dev/nvme0n1p1

# 根分区（ext4，4K 块）
mkfs.ext4 /dev/nvme0n1p3

# 或用 btrfs
mkfs.btrfs -L myarch /dev/nvme0n1p3 -S 4k

# swap
mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2

# 挂载
mount /dev/nvme0n1p3 /mnt
mount --mkdir /dev/nvme0n1p1 /mnt/boot
```

---

## NTFS 什么时候才用

只在 **Windows + Linux 双系统共享数据分区**时。

| 分区用途 | 推荐 |
|---|---|
| ESP（启动） | **FAT32**（强制） |
| 数据分区（双系统共享） | **NTFS** |
| Linux 独占分区 | ext4 |

只装 Arch，全 ext4。

---

## 一句话总结

**ESP 32K（默认），根分区 ext4 + 4K（默认），swap 4G**——你这配置的最优解。所有默认值都不用改。
