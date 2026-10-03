# Arch 装机全流程

## 两条路径

| 路径 | 命令 | 耗时 | 学到什么 | 适合 |
|---|---|---|---|---|
| **A. archinstall 向导** | `archinstall` | 20 分钟 | 知道每步在干嘛 | 先用起来再说 |
| **B. 手动 pacman 流程** ✅ | pacstrap + arch-chroot | 40-60 分钟 | 分区、文件系统、chroot、引导全链路 | 真正想懂 Linux 怎么装机 |

**底层一样**。A 是 B 的自动化工具。

---

## 前置：在 Windows 上准备镜像

### 1. 下载 ISO

`https://archlinux.org/download`

### 2. 验签（可选但官方建议）

```bash
curl -O https://archlinux.org/download/archlinux-2026.10.01-x86_64.iso
curl -O https://archlinux.org/download/sha256sums.txt
sha256sum -c sha256sums.txt
```

### 3. 写入 USB（Rufus 配置）

| 选项 | 设置 |
|---|---|
| 设备 | U 盘 |
| 引导类型 | **ISO 镜像** |
| 镜像文件 | `archlinux-2026.10.01-x86_64.iso` |
| 分区方案 | **GPT**（UEFI 机器） |
| 目标系统 | **UEFI** |
| 文件系统 | **FAT32** |
| 簇大小 | **32K（默认）** |
| 持久分区 | **不勾 / 设 0**（Arch 是安装型镜像，不是 Live USB） |

> **持久分区是什么**：Rufus 给 Live USB 用的功能（Ubuntu 试用），Arch ISO 不需要。

> **安全启动**：Arch ISO 不支持 Secure Boot，装机时要在 BIOS 里关掉。

---

## 手动安装全流程（路径 B）

从 USB 启动后按顺序执行。

### 1. 确认启动模式

```bash
cat /sys/firmware/efi/fw_platform_size
```

- 输出 `4` → UEFI（现代机器）
- 无输出 → BIOS/Legacy

决定后面 ESP 挂载点：UEFI 用 `/boot`，BIOS 用 `/boot/efi`。

### 2. 连网 + 时区

```bash
ping -c 3 archlinux.org
timedatectl set-ntp true              # 必须同步时间，否则包验证会失败
loadkeys us                           # 键盘布局
```

无线：`ip link` 看网卡名（如 `wlp2s0`），`wifi-menu` 或 `nmcli device wifi connect <SSID> password <密码>`。

### 3. 分区

```bash
lsblk                                # 看清楚盘符，别认错盘！
fdisk /dev/nvme0n1                   # 或 cgdisk（GPT 交互友好）
```

**240G 推荐分区**（UEFI）：

| 分区 | 大小 | 类型 | 用途 |
|---|---|---|---|
| `/dev/nvme0n1p1` | 512M | EFI System | ESP，放引导文件 |
| `/dev/nvme0n1p2` | 4G | Linux Swap | 备用（16G 内存不需要大 swap） |
| `/dev/nvme0n1p3` | ~235G | Linux root | `/`，根分区 |

fdisk 里 `n` 新建 → `t` 改类型（ESP 选 `ef`）→ `w` 写入。

### 4. 格式化 + 挂载

```bash
mkfs.fat -F32 /dev/nvme0n1p1          # ESP 必须 FAT32
mkswap /dev/nvme0n1p2 && swapon /dev/nvme0n1p2
mkfs.ext4 /dev/nvme0n1p3

mount /dev/nvme0n1p3 /mnt             # 根分区挂到 /mnt
mount --mkdir /dev/nvme0n1p1 /mnt/boot   # UEFI：ESP 挂到 /boot
```

### 5. 选国内镜像

```bash
reflector -a 20 --sort rate --save /etc/pacman.d/mirrorlist
```

或手动放清华/中科大/阿里源。

### 6. 安装基础包

```bash
pacstrap -K /mnt base linux linux-firmware
```

`-K` 保留内核模块，装内核时不重新编译。`base` 是唯一强制包组。

### 7. 生成 fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

`-U` 用 UUID，比 `/dev/sdX` 稳定（盘符会漂移）。

### 8. 进入新系统（chroot）

```bash
arch-chroot /mnt
```

从这里起所有操作都在新装的系统里了。

### 9. 时区 + 本地化

```bash
ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc

echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
sed -i 's/#en_US/en_US/' /etc/locale.gen
locale-gen
echo "LANG=zh_CN.UTF-8" > /etc/locale.conf
echo "KEYMAP=cn" > /etc/vconsole.conf
```

### 10. 主机名 + root 密码

```bash
echo "myarch" > /etc/hostname
passwd
```

### 11. 引导加载器（二选一）

**systemd-boot**（UEFI 推荐，简单）：

```bash
bootctl install
editor /boot/loader/loader.conf       # 写 default、timeout
editor /boot/loader/entries/arch.conf # 写 linux/initrd 行
```

**GRUB**（BIOS/UEFI 都能用）：

```bash
pacman -S grub os-prober
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

### 12. 生成 initramfs（关键，漏了进不去系统）

```bash
mkinitcpio -P
```

### 13. 退出 + 重启

```bash
exit
umount -R /mnt
reboot
```

---

## archinstall 向导（路径 A 速览）

从 USB 启动后：

```bash
archinstall
```

会引导你走：语言 → 磁盘 → 分区 → 内核 → 时区 → 用户 → 包 → 完成。

适合"先用起来再说"，但会跳过分区→格式化→挂载→chroot→引导的完整链路。

---

## 装完后立刻验证

```bash
uname -a                              # 内核
df -h                                 # 分区，/ 应 ~235G
cat /etc/os-release                   # 是 Arch Linux
ping -c 3 archlinux.org               # 网络
timedatectl                           # 时区 Asia/Shanghai
```

全绿就进 `04-post-install.md`。
