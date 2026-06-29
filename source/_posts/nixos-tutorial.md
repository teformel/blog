---
title: NixOS 极简生存指南：从安装、管理到 Flake 工具链使用
date: 2026-06-29 17:03:00
tags: [NixOS, Linux, Flake]
categories: 教程
---

本文旨在为初学者及进阶用户提供一份清晰、系统的 NixOS 使用教程。内容涵盖系统的安装与卸载、版本更新操作，以及现代 Flake 生态下常用效率工具的配置与使用方法。

---

## 1. NixOS 的安装与卸载

NixOS 拥有两种配置管理模式：**传统单文件模式（Non-Flake）** 和 **现代声明式模式（Flake）**。

下面您可以选择对应的模式查看具体指南：

{% tabs install_guide, 1 %}

<!-- tab 🛠️ 传统模式 (Non-Flake) -->

### 1.1 传统模式安装（从零开始）
如果您是第一次安装 NixOS，且不打算直接使用 Flake，推荐使用官方的 **NixOS Graphical Installer**（图形化安装镜像）。

1. **准备与启动**：
   - 前往 [NixOS 官网下载页](https://nixos.org/download) 下载带有 GNOME 或 KDE 桌面环境的 ISO 镜像。
   - 使用 Ventoy 或 Rufus 将 ISO 写入 U 盘并引导启动。
2. **运行安装器**：
   - 运行桌面上的 **Install NixOS**（基于 Calamares 安装器）。
   - 按照向导选择时区、分区（推荐选择自动抹除并分区，且启用 Swap 交换空间）、键盘布局和桌面环境。
   - 安装器在结束前，会自动在 `/etc/nixos/` 目录下生成两个最基础的配置文件：
     - `configuration.nix`：系统级全局配置（软件包、服务、网络等）。
     - `hardware-configuration.nix`：硬件驱动和分区挂载信息（通常无需手动修改）。
3. **重启**：安装完成后重启，拔掉 U 盘即可。

### 1.2 传统非 Flake 模式下的更新与回滚
* **日常更新**：
  ```bash
  sudo nix-channel --update
  sudo nixos-rebuild switch --upgrade
  ```
* **版本回滚**：
  ```bash
  sudo nixos-rebuild switch --rollback
  ```

<!-- endtab -->

<!-- tab 🚀 现代 Flake 模式 -->

### 2.1 现代 Flake 模式安装（已有配置文件）
如果您已经拥有一套保存在 Git 仓库中的 NixOS Flake 配置文件，想在全新机器上快速部署：

1. **引导并分区**：
   - 使用 Live CD 启动进入终端，对目标硬盘分区。以下是 UEFI 引导的推荐方案：
     ```bash
     # 假设硬盘为 /dev/sda
     parted /dev/sda -- mklabel gpt
     parted /dev/sda -- mkpart ESP fat32 1MiB 512MiB
     parted /dev/sda -- set 1 esp on
     parted /dev/sda -- mkpart primary ext4 512MiB 100%

     # 格式化
     mkfs.vfat -F 32 -n boot /dev/sda1
     mkfs.ext4 -L nixos /dev/sda2

     # 挂载分区到 /mnt
     mount /dev/disk/by-label/nixos /mnt
     mkdir -p /mnt/boot
     mount /dev/disk/by-label/boot /mnt/boot
     ```
2. **生成目标硬件配置**：
   ```bash
   nixos-generate-config --root /mnt
   ```
   这会在 `/mnt/etc/nixos/hardware-configuration.nix` 生成适合当前物理机的硬件配置文件。
3. **克隆配置并组合**：
   - 将您的 Flake 配置文件目录克隆到 `/mnt/etc/nixos/`（或任意临时目录）。
   - 将刚才生成的 `/mnt/etc/nixos/hardware-configuration.nix` 文件替换/放入您 Flake 配置中对应主机的硬件配置路径中。
4. **执行 Flake 安装**：
   使用 `--flake` 参数指定配置路径以及在 `flake.nix` 中定义的对应主机 Hostname：
   ```bash
   nixos-install --root /mnt --flake /mnt/etc/nixos#my-host
   ```
5. **设置密码与重启**：输入 root 密码后重启即完成部署。

### 2.2 现代 Flake 模式下的更新与回滚
* **日常更新**（会更新 `flake.lock` 文件）：
  ```bash
  nix flake update
  sudo nixos-rebuild switch --flake .#your-hostname
  ```
* **回滚命令**：
  ```bash
  sudo nixos-rebuild switch --rollback
  ```

<!-- endtab -->

{% endtabs %}

---

### 1.3 NixOS 的卸载与清理

NixOS 系统的卸载方式取决于您的目标是“彻底抹除”还是“安全回滚”。

#### 1.3.1 彻底卸载（更换回其他系统）
由于 NixOS 所有核心组件及依赖都保存在 `/nix/store` 中，且配置保存在单一目录，卸载非常干净：
1. **备份数据**：备份您的 `/etc/nixos` 目录或您的 Flake 代码库。
2. **格式化分区**：
   - 插入 Windows 或其他 Linux 系统的启动盘。
   - 打开分区工具直接删除/格式化 NixOS 所在的 `ext4`/`btrfs`/`zfs` 分区。
3. **清理 EFI 引导**：
   - 挂载您的 EFI 分区。
   - 删除路径下的 `EFI/nixos` 以及 `EFI/systemd`（如果使用的是 systemd-boot 引导）目录。这一步将彻底清除主板启动项中的 NixOS。

#### 1.3.2 历史包垃圾回收（Garbage Collection）
如果您只想释放磁盘空间：
```bash
# 删除所有超过 7 天的历史版本
sudo nix-env --delete-generations +7d --profile /nix/var/nix/profiles/system
# 执行真正的垃圾回收，释放磁盘空间
sudo nix-store --gc
```

---

## 2. 现代 NixOS Flake 必备工具链

为了极大提升 NixOS 的使用便利度，以下是几款社区公认“装机必备”的辅助工具：

### 2.1 极简重建与清理工具：`nh` (Nix Helper)

`nixos-rebuild` 命令较长且输出繁杂，`nh` 是一个更加优雅的系统重构替代命令行，提供了彩色的进度条 and 极为简便的清理指令。

#### 2.1.1 启用配置
在 `configuration.nix` 中加入：
```nix
programs.nh = {
  enable = true;
  clean.enable = true;
  clean.extraArgs = "--keep-since 4d --keep 3"; # 自动垃圾回收：自动保留最近4天的版本
  flake = "/home/username/dotfiles"; # 指向你默认的 Flake 配置所在绝对路径
};
```

#### 2.1.2 常用命令
* **一键切换系统配置**：`nh os switch`
* **测试编译配置**：`nh os test`
* **清理历史垃圾**：`nh clean all`

---

### 2.2 运行非 Nix 原生二进制程序：`nix-ld`

NixOS 独特的目录结构导致直接从网上下载的预编译二进制运行时会报错 `No such file or directory`。`nix-ld` 可以为系统伪造一个动态链接库加载路径。

#### 2.2.1 启用配置
在 `configuration.nix` 中启用，并提供常用动态库依赖：
```nix
programs.nix-ld.enable = true;
programs.nix-ld.libraries = with pkgs; [
  stdenv.cc.cc
  openssl
  uuid
  zlib
  glib
  libglvnd
  xorg.libX11
  xorg.libXcursor
  xorg.libXrandr
  xorg.libXi
];
```

---

### 2.3 零配置快速开发环境：`direnv` + `nix-develop`

在 NixOS 中，利用 `direnv` 可以在你进入某个项目目录时，自动在当前 shell 中加载该项目专属的软件依赖。

#### 2.3.1 启用配置
在 `configuration.nix` 中全局启用 `direnv`：
```nix
programs.direnv.enable = true;
programs.direnv.nix-direnv.enable = true;
```

#### 2.3.2 项目中使用
1. 在项目目录下新建 `flake.nix`：
   ```nix
   {
     inputs.nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
     outputs = { self, nixpkgs }:
       let
         pkgs = nixpkgs.legacyPackages.x86_64-linux;
       in {
         devShells.x86_64-linux.default = pkgs.mkShell {
           buildInputs = with pkgs; [ nodejs_20 python3 git ]; # 依赖
         };
       };
   }
   ```
2. 在该目录下新建 `.envrc` 并写入：`use flake`。
3. 执行 `direnv allow` 即可。

---

### 2.4 快速搜索与临时执行命令：`nix-index` / `comma`

使用 `nix-index` 配合 `comma`，系统可以直接临时拉取运行某个未安装的命令，无需将其写入配置文件。

#### 2.4.1 安装配置
在 `configuration.nix` 中加入：
```nix
programs.nix-index.enable = true;
```
并在 Flake 中将 `pkgs.comma` 加入 `environment.systemPackages`。

#### 2.4.2 使用方法
```bash
, neofetch
```
*(临时拉取运行 `neofetch`，结束后自动销毁)*
