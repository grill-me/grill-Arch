# grill-Arch

Arch Linux 装机 + 配置学习路径。

## 文件索引

| 文件 | 内容 |
|---|---|
| `01-arch-overview.md` | Arch 是什么、哲学、和其他发行版的区别 |
| `02-installation.md` | 装机全流程（手动 pacman，含 archinstall 对比） |
| `03-partition-and-fs.md` | 分区方案、文件系统、簇/块大小决策表 |
| `04-post-install.md` | 装完后的初始化：本地化、国内源、桌面、工具链 |

## 当前机器配置

| 项 | 值 |
|---|---|
| 内存 | 16G |
| 硬盘 | 240G NVMe |
| 目标用途 | 开发机（Node/Python/Java + ComfyUI） |

## 最终方案速查

```
nvme0n1 (240G)
├── p1  512M   ESP    → FAT32, 簇 32K
├── p2  4G     Swap   → Linux swap
└── p3  ~235G  Root   → ext4, 块 4K
```

桌面：KDE Plasma（首选）或 Sway（轻量 tiling）

## 相关

- IDEA.md：本仓库定位
- grill-nodeJS/：Node.js 学习路径（同系列）
