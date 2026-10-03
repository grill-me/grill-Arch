# 装机后初始化

按顺序走，每步可独立执行。

---

## Phase 0：验证安装成功（5 分钟）

```bash
uname -a                              # 看内核
df -h                                 # 看分区，/ 应 ~235G
cat /etc/os-release                   # 确认 Arch Linux
ping -c 3 archlinux.org               # 网络
timedatectl                           # 时区 Asia/Shanghai
```

全绿继续。

---

## Phase 1：本地化 + 中文（10 分钟）

```bash
# 键盘
echo "KEYMAP=cn" > /etc/vconsole.conf
loadkeys cn

# locale
locale -a | grep zh_CN.UTF-8          # 有就跳过
# 没有的话：
sed -i 's/#zh_CN.UTF-8 UTF-8/zh_CN.UTF-8 UTF-8/' /etc/locale.gen
locale-gen

echo "LANG=zh_CN.UTF-8" > /etc/locale.conf
export LANG=zh_CN.UTF-8
export LC_ALL=zh_CN.UTF-8
```

重启生效。

---

## Phase 2：国内镜像（关键，5 分钟）

默认源在国外，下载龟速。**先改源再继续**。

```bash
# 方法一：reflector 自动选
reflector --country China --sort rate --latest 20 --save /etc/pacman.d/mirrorlist

# 方法二：手动放清华源到最前面
sudo tee /etc/pacman.d/mirrorlist <<'EOF'
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.aliyun.com/archlinux/$repo/os/$arch
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
EOF
```

验证：`sudo pacman -Sy`，国内源应该 10MB/s+。

---

## Phase 3：包管理器基础（5 分钟）

### pacman 常用命令

```bash
sudo pacman -S <package>              # 安装
sudo pacman -Syu                      # 升级（每周跑）
sudo pacman -Qi <package>             # 查包信息
pacman -Ss <keyword>                  # 搜索
sudo pacman -Rns <package>            # 卸载 + 依赖清理
sudo pacman -Scc                      # 清缓存（每月）
```

### AUR（Arch User Repository）

官方源没有的包，走 AUR。**推荐用 paru**：

```bash
git clone https://aur.archlinux.org/paru.git
cd paru && makepkg -si
cd .. && rm -rf paru

# 用 paru 装任何 AUR 包
paru -S <package>
```

---

## Phase 4：桌面环境（10-20 分钟）

### 选项 A：KDE Plasma（推荐，开箱即用）

```bash
sudo pacman -S plasma kde-applications sddm
systemctl enable sddm.service
systemctl set-default graphical.target
reboot
```

**优点**：完整桌面、中文好、和 Windows 体验最接近
**缺点**：吃资源（idle ~800MB）

### 选项 B：Sway（轻量 tiling，键盘驱动）

```bash
sudo pacman -S sway swaybg swaylock swayidle wl-clipboard waybar dmenu
systemctl enable multi-user-graphical.target
# 编辑 ~/.config/sway/config
```

**优点**：省资源（idle ~200MB）、快、键盘党爽
**缺点**：要学键盘操作

### 选项 C：不装桌面，纯命令行开发机

只用 SSH + VS Code Remote 时可以跳过。**但跑 ComfyUI/浏览器需要 GUI**。

---

## Phase 5：开发者工具链

### Node.js

```bash
sudo pacman -S nodejs npm pnpm
node -v    # v24.x

# 或用 fnm 管理多版本
curl -fsSL https://fnm.vercel.app/install | bash
fnm install 24
fnm use 24
```

### Python + RAG 环境

```bash
sudo pacman -S python python-pip uv
uv --version
```

### Java（Android/Compose）

```bash
sudo pacman -S jdk-openjdk jdk8-openjdk
# 或给 AGP 8.7.2 用 JDK 17+：
sudo pacman -S jdk17-openjdk
```

> AGP 8.7.2 必须设 `org.gradle.java.home` 指向 JDK17+，否则 D8 dexmerger 崩。

### Android SDK

```bash
sudo pacman -S android-tools gradle
paru -S android-studio               # Android Studio 通过 AUR
```

### Git + VS Code

```bash
sudo pacman -S git code
```

### ComfyUI（NVIDIA GPU）

```bash
sudo pacman -S nvidia-dkms nvidia-utils
```

---

## Phase 6：迁移项目

你的项目都指向 `http://43.153.148.187:3000/api`，路径不用改。

```bash
cd ~/
git clone https://github.com/yoy-aww/mall-server.git
git clone https://github.com/yoy-aww/mall-manage.git
# mall-web, mall-flutter, mallRN 按需
```

---

## Phase 7：日常操作

| 频率 | 命令 | 作用 |
|---|---|---|
| 每周 | `sudo pacman -Syu` | 滚动更新 |
| 每月 | `sudo pacman -Scc` | 清缓存 |
| 装新工具 | `paru -S <pkg>` | 优先 AUR |
| 卸载 | `sudo pacman -Rns <pkg>` | -n 清配置 |
| 查包信息 | `paru -Qi <pkg>` | 看版本、依赖 |

---

## 建议执行顺序

```
1. Phase 0 验证 → Phase 2 改源（先做这两个，后面都要用）
2. Phase 1 中文 → Phase 3 pacman/AUR 基础
3. Phase 4 选桌面（KDE 还是 Sway，影响后续体验）
4. Phase 5 按手头项目装工具链（先 Node/Python，Java/Android 用到再说）
5. Phase 6 迁项目，Phase 7 日常
```
