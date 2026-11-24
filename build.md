# Redmi Note 10 Pro 5G（chopin / MT6891）Linux ROM 构建方案 —— denial 桌面

- **产物**：`boot.img`（主线内核）+ `linux-rootfs.img`（Arch Linux ARM + denial），fastboot 刷入 `boot` / `userdata` 分区
- **内核**：[MT6891-Mainline/linux](https://github.com/MT6891-Mainline/linux) `7.3.3-mt6893-xiaomi-chopin` 分支（主线 **Linux 7.3.3** 移植，`chopin_defconfig`，DTS `mt6893-xiaomi-chopin.dts`；构建时自动拉取，本仓库不携带内核源码）
- **桌面**：denial（官方 aarch64 Pacman 仓库二进制包，无需编译）
- **工具链**：GCC 交叉编译（`gcc-aarch64-linux-gnu`），**全程不使用 Clang**
- **执行方式**：全程**前台、单线程**（`-j1`），不加 `&` / `nohup` / `parallel`
- **CI 实现**：本仓库 `.github/workflows/build.yml`（独立 CI/CD，无 inputs，内核与桌面运行时自动拉取）；原 4.14 参考工作流项目已删除，差异对照见附录 B

---

## 1. 架构与流程总览

```
构建机 Ubuntu 24.04 x86-64（前台 / 单线程）
   │
   ├─ 阶段 0  构建机准备（GCC 交叉工具链、mkbootimg、fastboot）
   │
   ├─ 阶段 1  编译内核 linux/（7.3.3, chopin_defconfig）
   │           → Image.gz + mt6893-xiaomi-chopin.dtb + 内核模块
   │
   ├─ 阶段 2  制作 rootfs：Arch Linux ARM aarch64
   │           └─ 安装 denial（官方 aarch64 Pacman 仓）+ 开机自启（mobile 会话）
   │
   ├─ 阶段 3  内核模块并入 rootfs（modules_install）
   │
   ├─ 阶段 4  制作 initramfs（busybox，挂载 rootfs 后 switch_root）
   │
   ├─ 阶段 5  打包：mkbootimg → boot.img；ext4 → linux-rootfs.img
   │
   └─ 阶段 6  刷机：fastboot flash boot / userdata
```

**决定流程形态的内核事实**（来自 `linux/arch/arm64/configs/chopin_defconfig`，已核实）：

| 事实                                                             | 后果                                                                       |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `CONFIG_CMDLINE_FORCE=y`，且固化 cmdline 中**没有 `root=`**      | bootloader 无法指定根分区，**boot.img 必须携带 initramfs** 由其挂载 rootfs |
| `CONFIG_MODULES=y`，Wi-Fi 等大量驱动 `=m`                        | **必须 `modules_install`** 把 `/lib/modules/<版本>` 装进 rootfs            |
| `CONFIG_DRM_MEDIATEK=y`、chopin 面板驱动 `=y`（内建）            | 显示通路不依赖模块，denial 可直接出画面                                    |
| 串口 console 固化为 `console=ttyS0,921600n8`，`earlycon=mtk8250` | 首启排障可用串口日志                                                       |

---

## 2. 构建目录约定

以下命令假设在 Linux 构建机上按此布局（内核源码 = 全新 clone：`git clone --depth 1 -b 7.3.3-mt6893-xiaomi-chopin https://github.com/MT6891-Mainline/linux.git ~/chopin-rom/linux`）：

```
~/chopin-rom/
├── linux/                # 内核源码（Linux 7.3.3 移植树）
└── rom/
    ├── rootfs/           # Arch Linux ARM 根文件系统（阶段 2）
    ├── initramfs/        # busybox initramfs 组装目录（阶段 4）
    ├── initramfs.cpio.gz
    ├── boot.img
    └── linux-rootfs.img
```

---

## 3. 阶段 0：构建机准备（Ubuntu 24.04 x86-64）

```bash
sudo apt update
sudo apt install -y \
  git curl wget zip unzip build-essential bc bison flex \
  libssl-dev libelf-dev python3 python3-pip \
  gcc-aarch64-linux-gnu g++-aarch64-linux-gnu binutils-aarch64-linux-gnu \
  qemu-user-static cpio e2fsprogs xz-utils \
  android-tools-adb android-tools-fastboot

# mkbootimg（PyPI 版，提供 /usr/local/bin/mkbootimg 命令）
pip3 install mkbootimg
# 备选：从 AOSP 取脚本
#   https://android.googlesource.com/platform/system/tools/mkbootimg/
```

**不需要 Clang**：主线 7.x 内核用 GCC 交叉工具链即可（build.yml 用 Clang 是 4.14 安卓供应商内核的惯例，不适用于本方案）。denial 走官方二进制包，同样无编译器依赖。

---

## 4. 阶段 1：编译内核

```bash
cd ~/chopin-rom/linux

# 4.1 载入板级配置（主线树无 vendor/ 前缀；build.yml 的 4.14 树才是 vendor/chopin_defconfig）
make O=out ARCH=arm64 chopin_defconfig

# 4.2 编译（前台、单线程）
make -j1 O=out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-

# 4.3 安装模块到 rootfs（阶段 2 建好 rootfs 后执行，见阶段 3）
# make -j1 O=out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
#   INSTALL_MOD_PATH=$HOME/chopin-rom/rom/rootfs modules_install
```

> 不需要 `CROSS_COMPILE_ARM32`（那是 4.14 供应商内核的 32 位辅助代码用的）；不需要 KernelSU（安卓 root 方案，Linux ROM 用不上）。

**产物：**

| 产物     | 路径                                                        | 去向                      |
| -------- | ----------------------------------------------------------- | ------------------------- |
| 内核镜像 | `out/arch/arm64/boot/Image.gz`（若无则 `Image`）            | boot.img 的 `--kernel`    |
| 设备树   | `out/arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin.dtb` | boot.img 的 `--dtb`       |
| 内核模块 | `out/modules/lib/modules/<内核版本>/`                       | rootfs 的 `/lib/modules/` |

单线程全量编译耗时以小时计，属预期。

---

## 5. 阶段 2：制作 rootfs（Arch Linux ARM aarch64）并安装 denial

**选型说明**：denial 的 Alpine APK 包**只有 x86-64**（官方 `docs/INSTALL.md` 明确）；aarch64 包发布在与 x86-64 相同的 **Pacman / APT / DNF** 仓库中。选 Arch Linux ARM（Pacman 为 denial 第一方通道，`install.sh` 可识别 `ID=archarm` 自动配置）。

### 5.1 解压基础系统并进入 chroot

```bash
mkdir -p ~/chopin-rom/rom/rootfs && cd ~/chopin-rom/rom
wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
sudo tar -xzf ArchLinuxARM-aarch64-latest.tar.gz -C rootfs

# QEMU 用户态模拟，进入 ARM64 chroot（前台交互）
sudo cp /usr/bin/qemu-aarch64-static rootfs/usr/bin/
sudo mount --bind /proc     rootfs/proc
sudo mount --bind /sys      rootfs/sys
sudo mount --bind /dev      rootfs/dev
sudo mount --bind /dev/pts  rootfs/dev/pts
sudo chroot rootfs /bin/bash -l
```

### 5.2 chroot 内：初始化系统

```bash
pacman-key --init && pacman-key --populate archlinuxarm
pacman -Syu

ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
passwd                       # 设 root 密码
useradd -m -G wheel,audio,video,input,render user && passwd user

# 基础服务（手机无图形控制台，SSH 是首启后的主要排障通道）
pacman -S --needed udev networkmanager openssh e2fsprogs sudo curl gnupg
systemctl enable NetworkManager sshd

# 开机挂载根分区（LABEL 与阶段 5.2 打镜像时一致）
echo 'LABEL=linuxroot  /  ext4  defaults,noatime  0 1' >> /etc/fstab
```

### 5.3 安装 denial（官方 aarch64 Pacman 仓库）

```bash
# install.sh 依赖 sudo / curl / gnupg（上一步已装）；交互确认输 y
curl -fsSL https://install.denialwm.org -o setup.sh
sh setup.sh                  # 校验签名密钥指纹，写入 pacman.conf [denial] 段
pacman -Syu denial           # 连带安装 denial-flutter-engine
```

可选：手机专用软件仓 `https://packages.denialwm.org/phone/aarch64`（Pacman 格式，含 Phonewave/denial-apps 手机侧包），向 `pacman.conf` 追加 `[denial-phone]` 段并配置签名信任后使用。

### 5.4 开机自启 denial（手机 mobile 会话）

```bash
pacman -S greetd
```

`/etc/greetd/config.toml` 配置 `initial_session`（无人值守自动进入会话；会话命令以包内
`/usr/share/wayland-sessions/denial.desktop` 的 `Exec=` 为准）。核心环境与启动形态：

```bash
export DENIAL_SHELL_PROFILE=mobile          # 手机外壳配置
deniald --device /dev/dri/card0 \
        --output-config outputs.conf \
        --wayland --flutter-bundle <包内 bundle 路径>
# GPU 走独立 render node 时加：--render-device /dev/dri/renderD128
```

要求从具备 DRM / 输入设备权限的 seat 会话启动（greetd + logind 满足；`outputs.conf` 参考
`denial/packaging/arch/outputs.conf`）。

### 5.5 退出 chroot

```bash
exit   # 退出 chroot 后，宿主机执行：
sudo umount ~/chopin-rom/rom/rootfs/dev/pts \
             ~/chopin-rom/rom/rootfs/dev \
             ~/chopin-rom/rom/rootfs/proc \
             ~/chopin-rom/rom/rootfs/sys
sudo rm ~/chopin-rom/rom/rootfs/usr/bin/qemu-aarch64-static
```

---

## 6. 阶段 3：内核模块并入 rootfs

```bash
cd ~/chopin-rom/linux
make -j1 O=out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
  INSTALL_MOD_PATH=$HOME/chopin-rom/rom/rootfs modules_install
```

**版本一致性**：rootfs 内 `/lib/modules/<版本>` 必须来自**同一次**内核编译；重编内核后必须重新执行本步。

---

## 7. 阶段 4：制作 initramfs（busybox）

职责：cmdline 无 `root=` 且 FORCE，由 initramfs 按卷标找到 userdata 分区上的 ext4 → 挂载 → `switch_root`。

### 7.1 获取 aarch64 静态 busybox

```bash
# 方式一：官方预编译静态二进制
mkdir -p ~/chopin-rom/rom/initramfs/bin
wget -O ~/chopin-rom/rom/initramfs/bin/busybox \
  https://busybox.net/downloads/binaries/1.36.1-defconfig-multiarch-musl/busybox-aarch64
chmod +x ~/chopin-rom/rom/initramfs/bin/busybox

# 方式二（URL 失效时）：源码自编译（交叉 + 静态）
# tar -xjf busybox-1.36.1.tar.bz2 && cd busybox-1.36.1
# make defconfig
# make CROSS_COMPILE=aarch64-linux-gnu- LDFLAGS=--static -j1
# make CONFIG_PREFIX=<initramfs 目录> install
```

### 7.2 组装

```
initramfs/
├── init              # 可执行脚本（见下）
├── bin/busybox
├── newroot/          # 空目录，switch_root 挂载点
└── dev/ proc/ sys/   # 空目录
```

`init` 骨架（按需补全日志/重试）：

```sh
#!/bin/busybox sh
/bin/busybox mkdir -p /proc /sys /dev /newroot
/bin/busybox mount -t proc     proc     /proc
/bin/busybox mount -t sysfs    sysfs    /sys
/bin/busybox mount -t devtmpfs devtmpfs /dev
/bin/busybox mdev -s

ROOT=$(/bin/busybox blkid -L linuxroot)      # 与 fstab / mkfs.ext4 -L 一致
if [ -z "$ROOT" ]; then /bin/busybox sh; fi  # 找不到就进 shell 排查
/bin/busybox mount -o ro "$ROOT" /newroot || /bin/busybox sh
/bin/busybox switch_root /newroot /sbin/init
```

### 7.3 打包

```bash
chmod +x init
sudo find . | sudo cpio -o -H newc | gzip -1 > ../initramfs.cpio.gz
```

> 若内核移植仓库（MT6891-Mainline/linux 等）自带 initramfs 方案，**优先复用**，兼容性最好。

---

## 8. 阶段 5：打包镜像

### 8.1 boot.img

**mkbootimg 的分区参数（`--base` / `--kernel_offset` / `--pagesize` / header version）必须与设备一致，否则可能变砖。** 获取途径（按优先级）：

1. 内核移植仓库自带的打包脚本/参数说明（最可靠）；
2. 从小米官方线刷包提取原厂 `boot.img`，用 `magiskboot unpack` / `unpackbootimg` 读出实际参数后照抄。

命令模板：

```bash
mkbootimg \
  --kernel ~/chopin-rom/linux/out/arch/arm64/boot/Image.gz \
  --dtb    ~/chopin-rom/linux/out/arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin.dtb \
  --ramdisk ~/chopin-rom/rom/initramfs.cpio.gz \
  --header_version 2 \
  --base <按上方途径获取> --kernel_offset <同上> --pagesize <同上> \
  --output ~/chopin-rom/rom/boot.img
```

> `CONFIG_CMDLINE_FORCE=y`：cmdline 已固化进内核，mkbootimg 的 `--cmdline` 参数不参与启动，可不填。
> 刷前确认分区容量：`fastboot getvar partition-size:boot`。

### 8.2 linux-rootfs.img（ext4）

```bash
truncate -s 8G ~/chopin-rom/rom/linux-rootfs.img      # 大小按 rootfs 实际调整
mkfs.ext4 -L linuxroot ~/chopin-rom/rom/linux-rootfs.img

sudo mkdir -p /mnt/rootfs
sudo mount -o loop ~/chopin-rom/rom/linux-rootfs.img /mnt/rootfs
sudo cp -a ~/chopin-rom/rom/rootfs/. /mnt/rootfs/
sudo umount /mnt/rootfs

xz -T1 -9 ~/chopin-rom/rom/linux-rootfs.img           # 可选压缩；刷机前需 xz -d 还原
```

---

## 9. 阶段 6：刷机与首启

```bash
# 手机关机后按 Power + Vol Down 进 fastboot（或 adb reboot bootloader）
fastboot devices
fastboot flash boot     ~/chopin-rom/rom/boot.img
fastboot flash userdata ~/chopin-rom/rom/linux-rootfs.img   # 安卓数据全部清空
fastboot reboot
```

首启排障通道：

- **串口**：cmdline 已固化 `console=ttyS0,921600n8` + `earlycon=mtk8250`（需拆机接串口，开发者可选）；
- **SSH**：NetworkManager 起来后（USB 网络共享 / Wi-Fi 场景），用阶段 5.2 建的账户登录；
- 显示一黑到底但系统能 ping 通 → 查 denial 会话日志（journalctl）与 DRM 设备权限。

---

## 10. 回滚与备份（刷机前必做）

1. 从小米官方线刷包（对应 chopin 的 fastboot 包）提取 **原厂 `boot.img`、`dtbo.img`、`vbmeta.img`** 存档；
2. 回滚 = 原厂镜像回刷：
   ```bash
   fastboot flash boot   stock_boot.img
   fastboot flash dtbo   stock_dtbo.img
   fastboot flash vbmeta stock_vbmeta.img
   fastboot flash userdata stock_userdata.img   # 需要恢复安卓时
   fastboot reboot
   ```
3. `userdata` 被 Linux 系统整体占用：**安卓侧数据不可恢复**，刷机前自行迁移。

---

## 11. 单线程 / 前台执行约定（全程）

| 工具                                          | 常见并行写法 | 本方案写法                                        |
| --------------------------------------------- | ------------ | ------------------------------------------------- |
| make（内核 / modules_install）                | `-j$(nproc)` | `-j1`                                             |
| xz                                            | 自动多核     | `-T1`                                             |
| cpio / mkfs.ext4 / mkbootimg / fastboot / tar | —            | 本身单线程                                        |
| 所有命令                                      | —            | 不加 `&`、不用 `nohup` / `parallel`，前台阻塞执行 |

（附录 A 的 denial 源码编译路径中另有 `ninja -j 1`、`cargo --jobs 1`、`flutter --resource-pool-size=1`。）

---

## 12. 故障排查速查

| 症状                          | 处置                                                                                                                                                 |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| chroot 内 `pacman` 报签名错误 | 重跑 `pacman-key --init && pacman-key --populate archlinuxarm`；仍失败换 HTTPS 镜像源并更新 `archlinuxarm-keyring`                                   |
| 刷后不开机/卡 logo            | 串口抓日志；核对 mkbootimg 分区参数与 header version；回滚原厂 boot 排除硬件因素                                                                     |
| 内核起来但找不到根            | initramfs 进了 rescue shell → 手动 `blkid` 查卷标是否为 `linuxroot`；核对 mkfs 的 `-L` 与 init 脚本                                                  |
| rootfs 挂上但 systemd 起不来  | 多半 `/lib/modules` 与内核版本不一致 → 重做阶段 3                                                                                                    |
| denial 黑屏/闪退              | `journalctl` 查 deniald；确认 `/dev/dri/card0` 存在、会话有 video/input/render 权限、bundle 路径正确；必要时加 `--render-device /dev/dri/renderD128` |
| fastboot 刷 boot 报尺寸超限   | `fastboot getvar partition-size:boot` 核对；精简 initramfs（gzip -9 / 去掉多余 busybox applet）                                                      |

---

## 附录 A：denial 源码交叉编译（可选替代方案）

仅当需要修改 denial 源码时使用；完整步骤见 `denial/CROSS_BUILDING.md`，产物为 `denial-linux-arm64.tar.gz`（`bin/{deniald,denialctl,denial-portal}` + `bundle/`），解压进 rootfs。全部并行参数改为 1：

| 步骤            | 要点                                                                                                                                                                | 单线程参数               |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| 1. 前置         | `aarch64-linux-gnu-{gcc,g++,ar}` + ARM64 sysroot（DRM/EGL/GBM/libinput/libseat/udev/xkbcommon/Fontconfig/GTK3 开发库；**libc 须与 rootfs 一致，NDK sysroot 不行**） | —                        |
| 2. 依赖         | `tools/denial-pc bootstrap`、`tools/denial-flutter-engine prepare-app-build`、`rustup target add aarch64-unknown-linux-gnu`                                         | —                        |
| 3. Flutter 引擎 | `gn --runtime-mode=release --linux --linux-cpu=arm64 --enable-fontconfig ...` 后 ninja 构建目标集                                                                   | `ninja -j 1 <目标>`      |
| 4. 外壳 AOT     | `flutter assemble --local-engine=denial_linux_arm64_release ... -dTargetPlatform=linux-arm64`                                                                       | `--resource-pool-size=1` |
| 5. 合成器       | `cargo build --locked --release --target aarch64-unknown-linux-gnu --features flutter --bin deniald --bin denialctl` 与 `-p denial-portal`                          | `--jobs 1`               |
| 6. 组装         | 按 CROSS_BUILDING.md 第 8 步收集 payload → tar.gz（`file` 校验全部为 ARM aarch64）                                                                                  | —                        |

---

## 附录 B：与原 4.14 参考工作流（项目已删除）的对应关系

| 项        | build.yml（4.14 供应商内核）                | 本方案（主线 7.3.3 Linux ROM）                       |
| --------- | ------------------------------------------- | ---------------------------------------------------- |
| 目标      | 安卓内核替换（AnyKernel3 zip，recovery 刷） | 完整 Linux ROM（boot.img + rootfs.img，fastboot 刷） |
| 工具链    | Google Clang r547379                        | Ubuntu GCC 交叉工具链（无 Clang）                    |
| defconfig | `vendor/chopin_defconfig`                   | `chopin_defconfig`（主线树无前缀）                   |
| 并行      | `-j$(nproc)`                                | `-j1` 前台单线程                                     |
| KernelSU  | 有（安卓 root）                             | 无需                                                 |
| 产物      | `Kernel-Chopin-KernelSU.zip`                | `boot.img` + `linux-rootfs.img`                      |
| 桌面      | 安卓启动器                                  | denial（官方 aarch64 Pacman 包）                     |
