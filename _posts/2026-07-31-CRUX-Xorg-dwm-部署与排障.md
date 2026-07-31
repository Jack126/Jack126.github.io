---
layout: post
title: "CRUX Xorg dwm 部署与排障"
category: Linux
tags: CRUX Linux Xorg dwm vxwm
date: 2026-07-31 12:00
---

### 背景

CRUX Linux 3.8 虚拟机环境，从零编译 Xorg → 搭建 dwm/dmenu。完整实战记录，含 vxwm 尝试失败结论。

### ports 国内源与依赖安装

* CRUX 默认官方源国内不可用：修改 `/etc/ports/*.rsync` 换镜像 → `ports -u`
* 清华/USTC/阿里源各有坑，需逐个实测可用性

### 从零编译 Xorg

**前置**：
* 最小系统不带 X：`prt-get depinst xorg/xorg-xinit`（提供 startx）
* xorg 系列 port 统一在 `/usr/ports/xorg/`，包名带 **`xorg-` 前缀**（如 `xorg-libx11`）

**依赖链（严格顺序，踩坑记录）**：
```
xorg-libevdev → mtdev → libinput → xorg-xf86-input-libinput
```
libinput 编译必需 **libevdev + mtdev**，漏装直接 meson 报错。

**编译脚本核心模式**：
```
cd /usr/ports/xorg/libinput
pkgmk -d
OUT=$(pkgadd *.pkg.tar.* 2>&1)
if echo "$OUT" | grep -q "already installed"; then
  pkgadd -u *.pkg.tar.*
fi
```

### 完整版 build_xorg.sh

> 经过实战迭代的修正要点：
> 1. 依赖顺序严格排序：`xorg-libevdev → mtdev → libinput → xorg-xf86-input-libinput`
> 2. 目录名称严格匹配 `/usr/ports/xorg` 列表：`xorg-libevdev`（不是 libevdev）
> 3. 内置目录校验，不存在自动跳过
> 4. 自动判断 pkgadd 是否需要 `-u`
> 5. 关闭 `set -e`，单个包失败不中断整体流程
> 6. 脚本尾部清理 X 锁与授权文件，规避 `bad display name`

```bash
#!/bin/bash
set +e
PORTDIR="/usr/ports/xorg"

build_install() {
    local name="$1"
    echo -e "\033[32m==================== 正在处理 $name ====================\033[0m"
    if [ ! -d "${PORTDIR}/${name}" ]; then
        echo -e "\033[31m[警告] ${PORTDIR}/${name} 目录不存在，跳过\033[0m"
        return 0
    fi
    cd "${PORTDIR}/${name}"

    pkgmk -d
    PKG_FILE=$(ls *.pkg.tar.* 2>/dev/null)
    if [ -z "$PKG_FILE" ]; then
        echo -e "\033[31m[$name] 编译失败，未生成包文件，跳过\033[0m"
        cd "${PORTDIR}"
        return 0
    fi

    OUT=$(pkgadd "$PKG_FILE" 2>&1)
    if echo "$OUT" | grep -q "already installed"; then
        echo "[$name] 已安装，执行升级 pkgadd -u"
        pkgadd -u "$PKG_FILE"
    elif echo "$OUT" | grep -q "successfully installed"; then
        echo "[$name] 全新安装完成"
    else
        echo "$OUT"
    fi

    echo -e "\033[34m$name 处理结束\033[0m"
    cd "${PORTDIR}"
}

# 【重点】严格依赖顺序！不要乱调整顺序
BASE_LIBS=(
xorg-libevdev
mtdev
libinput
xorg-libx11
xorg-libxext
xorg-libxfixes
xorg-libxdamage
xorg-libxrender
xorg-libxcursor
xorg-libxinerama
xorg-libxrandr
xorg-libxtst
xorg-libxscrnsaver
xorg-libpixman
xorg-libxshmfence
xorg-libxcb
xorg-xcb-util
libdrm
mesa
xorg-xauth
xorg-xinit
xorg-server
xorg-xf86-input-libinput
xorg-sessreg
)

# 批量编译所有依赖
for pkg in "${BASE_LIBS[@]}"; do
    build_install "$pkg"
done

# ========== 显卡驱动选择 ==========
# 物理机Intel核显启用
#build_install xorg-xf86-video-intel
# VMware虚拟机请注释上一行，启用下面这行
build_install xorg-xf86-video-vmware
# VirtualBox
# build_install xorg-xf86-video-vboxvideo
# 通用兜底vesa
# build_install xorg-xf86-video-vesa

# 清理X残留锁、授权文件
echo -e "\033[32m清理Xorg残留文件\033[0m"
pkill -9 Xorg startx 2>/dev/null
rm -f /etc/X11/xorg.conf
rm -rf /etc/X11/xorg.conf.d
rm -rf /tmp/.X0-lock /tmp/.X11-unix/X0 ~/.Xauthority /root/.Xauthority

echo -e "\033[32m===== Xorg组件编译脚本执行完毕 =====\033[0m"
echo "启动图形操作："
echo "su - 普通用户名"
echo "startx"
echo "⚠️ 不要手动执行 xauth generate :0，会报错 unable to open display"
```

**使用方式**：
```
cd /usr/ports/xorg
nano build_xorg.sh       # 粘贴全部代码
chmod +x build_xorg.sh
./build_xorg.sh
```

**启动图形注意事项**：
* 禁止 root 执行 startx；切普通用户直接 `startx`（自动创建授权 cookie）
* 不要手动运行 `xauth generate :0 . trusted`
* 若某包编译报错：进入对应目录手动 `pkgmk`（不带 `-d`）查看完整编译日志，定位缺失依赖

### Xorg 启动排障

**1. `Failed to activate virtual core keyboard`（最高频）**，按概率排序：
1. **缺 `xkeyboard-config`**（日志 `XKB: Couldn't open rules file .../xkb/rules/evdev`）→ 装即修复
2. 缺 eudev → `/dev/input` 空，X 无法发现键盘（现代 Xorg AutoAddDevices 依赖 udev）
   * CRUX 包名 **eudev**（core 仓库），临时启动：`/sbin/udevd --daemon; udevadm trigger`
3. 输入驱动：`xorg-xf86-input-evdev` / `xorg-xf86-input-libinput`
4. 权限：`usermod -aG input 用户名`（重新登录生效）

**2. VMware 显卡**：
* 符号 **`CONFIG_DRM_VMWGFX`**（菜单 "DRM driver for VMware Virtual GPU"），依赖 `DRM && PCI && X86`。选中按"Y"，若是 M，则去外层选 Y，最终前面显示 * 即可
* 无 initramfs → `CONFIG_DRM=y` 与 `CONFIG_DRM_VMWGFX=y` 都须内置

**3. dbus 缺失**：
* `prt-get depinst dbus`；无 systemd 用 rc.d：`/etc/rc.d/dbus start` + `ln -s /etc/rc.d/dbus /etc/rc.d/runlevel.boot/`
* **dwm/vxwm 本身不依赖 DBus**，缺 DBus 不会导致 WM 键盘失灵

**4. `bad display name` / xauth 失败**：
* 多为 Xorg 未启动成功或残留锁：`pkill -9 Xorg; rm -rf /tmp/.X0-lock ~/.Xauthority`

### CRUX 包管理深坑（pkgmk/pkgadd）

| 场景 | 正确姿势 |
|------|---------|
| 查包是否安装 | `pkginfo -i 包名`（**不能裸写 `pkginfo 包名`**） |
| 安装本地 pkgmk 产物 | `pkgadd ./包名#版本-1.pkg.tar.gz`（**必须带 `./`**） |
| 修改 Pkgfile 后编译 | `pkgmk -is`（忽略签名）；长期 `pkgmk -us`（刷新签名） |
| 离线编译 | 源码 tar 复制到 **port 当前目录** → `pkgmk -is` → `pkgadd ./...`（pkgmk 不读 distfiles 目录） |

> pkgmk 参数以 `pkgmk -h` 为准：`-d` 下载源码 / `-is` 忽略签名 / `-us` 更新签名 / `-im` 忽略 md5 / `-i` 构建并安装 / `-u` 升级安装

**libev 离线安装完整实战**：

场景：`prt-get install libev` 时官方源 `dist.schmorp.de` 80 端口连不通（国内网络），本地已有 `/usr/ports/distfiles/libev-4.33.tar.gz`，需要离线编译。

关键规则（pkgmk 源码查找逻辑）：
* `source=` 带 http → prt-get 强制 curl 下载，**无视本地缓存**（prt-get 没有禁止下载的参数）
* `source=` 仅文件名 → pkgmk 只在**当前 port 目录**找源码，不读 `/usr/ports/distfiles`
* pkgmk 无 `-d 路径` 用法（`-d` 单参数 = 开启下载）；曾有错误建议 `pkgmk -d/usr/ports/distfiles` 会报 invalid option
* 修改过 Pkgfile → 内置签名失效 → `Signature mismatch found: MISMATCH Pkgfile`（`-im` 只忽略 md5，需用 `-is`）

**最终可用流程**：
```
cd /usr/ports/opt/libev
# 1. 拷贝源码包到当前port目录（pkgmk只会在当前目录找源码）
cp /usr/ports/distfiles/libev-4.33.tar.gz ./

# 2. 忽略签名校验进行构建
pkgmk -is

# 3. 安装本地生成的包（重点：必须带 ./ ）
pkgadd ./libev#4.33-1.pkg.tar.gz

# 4. 验证
pkginfo | grep libev
```

**备选规范做法（永久消除签名报错）**：
```
cd /usr/ports/opt/libev
cp /usr/ports/distfiles/libev-4.33.tar.gz ./
pkgmk -us          # 更新签名
pkgmk              # 正常构建，无需附加参数
pkgadd ./libev#4.33-1.pkg.tar.gz
```

**pkgadd 语法区分（连续踩坑点）**：
* ❌ `pkgadd libev` → "could not determine name and/or version"：这是从已注册仓库/数据库安装的写法，不适合本地刚打包的文件
* ✅ `pkgadd ./libev#4.33-1.pkg.tar.gz` → 安装当前目录下的本地包，**必须带路径 `./`**
* 卸载：`pkgrm libev`（不需要路径）

> 教训：`pkgmk -h` 帮助清单里没有 `-n`（曾有错误建议 `pkgmk -n` 关闭签名，实际无效）；正确忽略签名参数是 `-is`。离线场景统一流程：`pkgmk -is`（或先 `pkgmk -us` 刷新签名）→ `pkgadd ./包名#版本-1.pkg.tar.gz`。
> 教训2： libev没啥用，与dwm无关。

### dwm / dmenu 配置

**一键部署脚本（suckless 编译安装到 /usr）**：

> 功能：安装 X 基础依赖、编译 dmenu + dwm（统一安装到 `/usr/bin`，规避 PATH 问题，CRUX 规范路径）。普通用户执行，安装步骤需要输入 root 密码。

```bash
#!/bin/sh
# build_dwm.sh -- CRUX dwm + dmenu 一键部署脚本
# 推荐：标准 dwm（规避vxwm不稳定问题）

set -e

echo "===== 更新ports索引，安装X基础依赖 ====="
prt-get update
prt-get install \
xorg xorg-fonts fontconfig freetype \
mesa mesa-dri xf86-input-libinput \
git gcc make

# 显卡驱动按需自行取消注释
# prt-get install intel-dri
# prt-get install radeon-dri
# prt-get install nouveau-dri

echo "===== 创建源码目录 ~/src ====="
mkdir -p ~/src
cd ~/src

echo "===== 编译安装 dmenu (PREFIX=/usr) ====="
if [ ! -d dmenu ]; then
    git clone https://git.suckless.org/dmenu
fi
cd dmenu
git pull
make clean
su -c "make PREFIX=/usr install"
cd ..

echo "===== 编译安装 dwm (标准原版，替代不稳定vxwm) ====="
if [ ! -d dwm ]; then
    git clone https://git.suckless.org/dwm
fi
cd dwm
git pull
# 启用编译优化 O2
sed -i 's/CFLAGS += -std=c99 -Wall/CFLAGS += -std=c99 -Wall -O2/' Makefile
# 可在此处自定义 config.h
# cp config.def.h config.h
make clean
su -c "make PREFIX=/usr install"
cd ..

echo "===== 检查安装结果 ====="
which dwm
which dmenu
which dmenu_run
```

* 自定义逻辑全在 `config.h`，修改后**必须重新编译**
* dmenu 快捷键：`{ MODKEY, XK_p, spawn, {.v = "dmenu_run" } }`（调用 `dmenu_run`）
* **PATH 坑**：startx 会话不读 `~/.profile`/`~/.bashrc`，`.xinitrc` 顶部 `export PATH="/usr/local/bin:$PATH"`
* 壁纸：`prt-get depinst feh`，`.xinitrc` 中 `feh --bg-scale /path/bg.png &`

**推荐 `.xinitrc` 模板**：
```sh
#!/bin/sh
export PATH="/usr/local/bin:$PATH"
export _JAVA_AWT_WM_NONREPARENTING=1

xset r rate 250 40
xsetroot -cursor_name left_ptr
while true; do
    xsetroot -name " $(date '+%m-%d %H:%M') | $(uptime | awk '{print $3,$4,$5}' | sed 's/,//') "
    sleep 1
done &

exec dwm          # ❌ 禁止 exec dwm &；exec 必须最后一行
```

### 常用排障命令

```
grep EE ~/.local/share/xorg/Xorg.0.log    # X 错误日志
DISPLAY=:0 /usr/local/bin/vxwm            # 手动运行捕获崩溃原因
pkill -9 Xorg                             # ssh 远程杀卡死 X
glxinfo | grep direct                     # 验证硬件加速
pkginfo | grep -E '包名'                  # 查已安装包
pkgwhich /bin/bash                        # 查文件属主包
```

### sshd 加入开机启动

CRUX 原生使用 rc.d（不是 runit/svinit），sshd 开机自启：

```
vi /etc/rc.conf
SERVICES=(lo net crond sshd)

# 立即启动
/etc/rc.d/sshd start
```
