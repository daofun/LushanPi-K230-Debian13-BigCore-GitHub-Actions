# LushanPi K230 Debian 13 (Big Core) 自动编译

本仓库包含 GitHub Actions 工作流，用于自动编译 **立创庐山派 K230** 在大核（Big Core / CPU1）上运行 **Debian 13 (trixie)** 的完整 SD 卡镜像。

## 架构说明

```
┌─────────────────────────────────────────────┐
│  K230 SoC                                   │
│  ┌──────────────────┐  ┌──────────────────┐  │
│  │   Big Core       │  │  Little Core     │  │
│  │   CPU1 1.6GHz    │  │  CPU0 800MHz     │  │
│  │   RISC-V C908    │  │  RISC-V C906     │  │
│  │                  │  │                  │  │
│  │   Debian 13      │  │  RT-Smart        │  │
│  │   (Linux)        │  │  (Real-time OS)  │  │
│  │   ─────────      │  │  ──────────      │  │
│  │   U-Boot         │  │  MPP / Drivers   │  │
│  │   OpenSBI        │  │                  │  │
│  │   Linux Kernel   │  │                  │  │
│  └──────────────────┘  └──────────────────┘  │
│          │                        │          │
│          └──── Shared Memory ─────┘          │
└─────────────────────────────────────────────┘
```

> 本工作流编译的镜像让 **Debian 13 运行在大核（Big Core）** 上，小核运行 RT-Smart 处理实时外设。

## 工作流流程

### Job 1: 构建 Debian 13 根文件系统
1. 使用 `debootstrap` 通过 qemu 交叉引导 RISC-V 64 的 Debian trixie
2. 配置 apt 源、主机名、时区、网络
3. 安装板载所需软件包（SSH、WiFi、GPIO 工具等）
4. 生成 ext4 格式的根文件系统镜像

### Job 2: 构建 K230 SDK
1. 克隆 K230 Linux SDK
2. 使用 Docker 容器编译 U-Boot、OpenSBI、Linux Kernel
3. 替换根文件系统为 Debian 13
4. 修改启动参数（bootcmd）指向 Debian 根分区
5. 生成最终 SD 卡镜像

### Job 3: 发布 Release
- 仅当推送 tag 或手动触发时执行
- 自动创建 GitHub Release 并上传镜像文件

## 使用方法

### 方式一：GitHub Actions 手动触发

1. Fork 本仓库
2. 进入 **Actions** 选项卡
3. 选择 **Build K230 Debian 13 Image for LushanPi (Big Core)**
4. 点击 **Run workflow**
5. 可选参数：
   - `sdk_branch`: SDK 分支（默认 main）
   - `defconfig`: 板级配置（默认 k230_canmv_lckfb_defconfig）
   - `debian_mirror`: Debian 镜像源
   - `image_version`: 镜像版本标签
6. 等待编译完成（约 30-60 分钟）
7. 在 Artifacts 中下载编译产物

### 方式二：推送 Tag 自动发布

```bash
git tag v1.0.0
git push origin v1.0.0
```

会自动触发编译并创建 GitHub Release。

## 目录结构

```
.
├── .github/
│   └── workflows/
│       └── build-debian13.yml      # GitHub Actions 工作流
├── rootfs/                         # 自定义根文件系统文件（可选）
│   └── etc/
│       └── ...                     # 覆盖默认配置
├── patches/                        # SDK 补丁（可选）
│   └── ...
└── README.md
```

## 烧录方法

### Linux
```bash
# 确认 SD 卡设备节点
lsblk

# 烧录（替换 /dev/sdX 为实际设备）
sudo dd if=LushanPi-K230_Debian13_bigcore_*.img of=/dev/sdX bs=1M oflag=sync
sync
```

### Windows
使用 [Balena Etcher](https://www.balena.io/etcher/) 或 [Rufus](https://rufus.ie/) 烧录 .img 文件到 SD 卡。

## 登录信息

| 方式 | 参数 |
|------|------|
| 串口 | 115200 8N1，无流控 |
| 默认用户 | root |
| 默认密码 | root |
| SSH | `ssh root@<板子IP>` |

## 自定义根文件系统

在 `rootfs/` 目录下放需要覆盖的文件，工作流会自动将其复制到根文件系统中。例如：

```
rootfs/
├── etc/
│   ├── wifi_config.conf       # WiFi 配置
│   └── systemd/
│       └── system/
│           └── my-service.service
└── usr/
    └── local/
        └── bin/
            └── my-script.sh
```

## 相关链接

- [K230 Linux SDK](https://github.com/kendryte/k230_linux_sdk)
- [嘉楠开发者社区](https://www.kendryte.com/)
- [庐山派官方文档](https://wiki.lckfb.com/zh-hans/lushan-pi-k230/)
- [K230 Debian/Ubuntu 指南](https://www.kendryte.com/k230_linux/zh/main/app_develop_guide/user_develop/debian_ubuntu.html)
- [K230 Linux boot (Big Core Linux)](https://code.videolan.org/jdek/k230-boot)
