---
layout: post
title: "CRUX Linux 内核升级完整教程"
category: Linux
tags: CRUX Linux 内核 编译
date: 2026-07-31 12:00
---

### 背景

CRUX Linux 3.8（默认内核 6.12.23）升级内核到 7.0.14，在 VMware 虚拟机环境遇到依赖安装、内核编译配置、GRUB 引导、UEFI 固件、显卡驱动等系列问题，记录完整排查过程。

### 关键认知（CRUX 内核管理特点）

* CRUX **官方不打包分发内核**（已删除 `core/linux` port），内核源码不受 pkg 数据库管理，手动编译是唯一方式
* 源码放哪都行（`/usr/src` 只是惯例），替换/删除不影响系统包数据库
* **CRUX 默认无 initramfs** → 显卡驱动必须 `<*>` 内置编译进内核镜像，不能 `<M>` 模块，否则开机无法加载

### 下载源码

```
cd /usr/src
wget https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.0.14.tar.xz
tar xf linux-7.0.14.tar.xz
cd linux-7.0.14
```

### 配置内核

```
# 从当前运行内核导出配置（推荐）
zcat /proc/config.gz > .config

# 自动适配新版本新增选项
make olddefconfig

# 可选：按需修改内核功能
# make menuconfig
```

`make olddefconfig` 对新选项使用默认值，稳定省心。`make oldconfig` 会逐个询问新选项。

**手动修改 `.config` 后必须执行 `make olddefconfig`**：直接 `vim/sed .config` 后编译，依赖冲突会被静默覆盖（如 CONFIG_DRM=m 会强制 CONFIG_DRM_VMWGFX=m），且难以察觉。正确流程：改 `.config` → `make olddefconfig` → 编译。

### 编译并安装

```
# 多线程编译
make -j$(nproc) bzImage modules

# 安装模块（剥离调试信息）
make INSTALL_MOD_STRIP=1 modules_install

# 复制启动文件到 /boot
cp arch/x86_64/boot/bzImage /boot/vmlinuz-7.0.14
cp System.map /boot/System.map-7.0.14
cp .config /boot/config-7.0.14

# 更新模块依赖
depmod -a 7.0.14

# 重建 GRUB 引导
grub-mkconfig -o /boot/grub/grub.cfg
```

> 注意：bzImage 路径是 `arch/x86_64/boot/bzImage`（x86_64 架构），不要写成 32 位的 `arch/x86/boot/`。

### VMware 虚拟机特殊处理

**UEFI vs BIOS 固件**：CRUX GPT 分区 + EFI 安装必须使用 **UEFI 固件**，不能用传统 BIOS。

切换方法：
1. 完全关闭虚拟机
2. 虚拟机设置 → **选项** → **高级**
3. 固件类型从 "BIOS" 改为 **"UEFI"**

**进入 UEFI Shell**：右键虚拟机 → **电源** → **打开电源时进入固件**

手动启动 GRUB：
```
fs0:
cd EFI\CRUX
grubx64.efi
```

GRUB 命令行加载配置：`configfile /grub/grub.cfg`

**永久修复虚拟机引导**：编辑虚拟机 `.vmx` 文件，末尾添加：
```
firmware = "efi"
efiBootEntry = "EFI\CRUX\grubx64.efi"
```

### 旧内核清理

确认新内核稳定后：

```
# 删除旧模块
rm -rf /lib/modules/6.12.23

# 删除旧启动文件
rm -f /boot/vmlinuz-6.12.23 /boot/System.map-6.12.23 /boot/config-6.12.23

# 刷新 GRUB 菜单
grub-mkconfig -o /boot/grub/grub.cfg

# 删除旧源码（可选）
rm -rf /usr/src/linux-6.12.23
```

### 完整一键编译脚本

```
ports -u
prt-get depinst gcc make ncurses flex bison elfutils openssl bc dwarves zstd git

cd /usr/src/linux-7.0.14

zcat /proc/config.gz > .config
make olddefconfig
make -j$(nproc) bzImage modules
make INSTALL_MOD_STRIP=1 modules_install

cp arch/x86_64/boot/bzImage /boot/vmlinuz-7.0.14
cp System.map /boot/System.map-7.0.14
cp .config /boot/config-7.0.14

depmod -a 7.0.14
grub-mkconfig -o /boot/grub/grub.cfg
```

### 常见坑与内核配置

**重启后内核版本没变**：
* 多为 grub.cfg 未更新或引导顺序问题：`grub-mkconfig -o /boot/grub/grub.cfg`
* EFI 引导缺失时重建：
```
grub-install --target=x86_64-efi --efi-directory=/boot --boot-directory=/boot --bootloader-id=CRUX --recheck
```

**`arch/` 目录与 Arch Linux 无关**：
* 内核源码 `arch/` = **Architecture（架构）**，存放各 CPU 架构专属代码，所有发行版通用
* `arch/x86_64/boot/bzImage` = x86_64 架构编译生成的压缩内核镜像（bzImage = big zImage）
* 其他架构：ARM64 → `arch/arm64/boot/Image`，RISC-V → `arch/riscv/boot/Image`

**menuconfig 找不到菜单（如 Graphics support）**：
* 多为旧 `.config` 残留依赖关闭导致菜单被隐藏
* 排查：menuconfig 内按 `/` 搜索目标符号（如 `FB`、`DRM_VMWGFX`），会提示缺失依赖
* 根治：`make mrproper` 清空旧配置 → 重新导入 → `make olddefconfig` → `make menuconfig`

**VMware 虚拟机显卡（CONFIG_DRM_VMWGFX）**：
* 选项正式名：**DRM driver for VMware Virtual GPU**（不是 "VMware SVGA"）
* 依赖链：`CONFIG_DRM_VMWGFX depends on DRM && PCI && X86`，DRM 未开则菜单直接隐藏
* **CRUX 无 initramfs → `CONFIG_DRM=y` 与 `CONFIG_DRM_VMWGFX=y` 必须都 `<*>` 内置**
* 依赖项为 `<M>` 时被依赖项无法设为 `<*>`（依赖是模块则被依赖只能模块），需先改 DRM 为 `<*>`
* 直接改 `.config` 方式（配合 olddefconfig）：
```
cd 内核源码目录
sed -i 's/CONFIG_DRM=m/CONFIG_DRM=y/' .config
sed -i 's/# CONFIG_DRM is not set/CONFIG_DRM=y/' .config
sed -i 's/CONFIG_DRM_VMWGFX=m/CONFIG_DRM_VMWGFX=y/' .config
sed -i 's/# CONFIG_DRM_VMWGFX is not set/CONFIG_DRM_VMWGFX=y/' .config

# 重中之重，自动处理依赖冲突！
make olddefconfig

# 校验是否生效
grep -E 'CONFIG_DRM=|CONFIG_DRM_VMWGFX=' .config
# 正确输出：CONFIG_DRM=y 与 CONFIG_DRM_VMWGFX=y
```

**手动改 `.config` vs make menuconfig**：
* `.config` 是纯文本，`sed`/编辑器直接改语法可行，但**依赖不会自动处理**
* 直接 `make` 不会修正冲突，会在编译阶段静默降级（想设 `<*>` 结果变 `<M>`）
* 正确顺序：改 `.config` → `make olddefconfig` → （可选 menuconfig 确认）→ `make`
* 首次调整互相依赖的选项组建议打开一次 menuconfig 确认状态；后续小修改可用 sed + olddefconfig
