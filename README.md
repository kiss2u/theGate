# 🛡️ Gateway-Warrior ISO Automated Github CI & Test Runner

本项目提供了一个基于 **GitHub Actions (x86_64 原生环境)** 的完全隔离、零环境污染的自动化编译、监控与测试部署解决方案。

整个流水线完全在 GitHub 云端沙盒中运行，最终输出适用于 **PVE 9.2.11** 软路由网关的定制版内存运行镜像 (`gateway-warrior.iso`)。本地开发机不需要安装任何编译依赖。

---

## 🏗️ 1. 系统网络架构与端口解耦
- **53 端口 (AdGuard Home)**: 应用层强力去广告，国内 DNS 完美秒开。
- **内核 TC 层 (xdns-bpf)**: 纳秒级 FNV-1a 哈希匹配黑名单。仓库: https://github.com/liudf0716/xdns-bpf
- **tun0 虚拟网卡 (xray-lite)**: 仓库: https://github.com/undead-undead/xray-lite 。盲抓内核重定向进来的 TCP 流量，VLESS + REALITY 加密出境，100% 保持三层原始目标 IP 完好无损。
- **持久化层 (/dev/vda)**: 自动格式化并挂载 PVE 本地虚拟磁盘，用于存储节点、黑名单和去广告缓存。固件本身保持绝对只读 (Run-from-RAM)。

---

## 📁 2. 仓库根目录结构

```text
你的GitHub代码仓库/
├── .github/
│   └── workflows/
│       └── build-iso.yml
├── configs/
│   ├── gateway_defconfig      <-- Buildroot defconfig
│   └── kernel.fragment        <-- 内核 eBPF/TC/TUN 补丁片段
├── overlay/
│   └── etc/
│       └── init.d/
│           └── S99gateway     <-- 引导脚本 (S99 级别最后拉起)
└── README.md
```

所有二进制依赖在 CI 内从源码编译或从官方 Release 下载，**无需在仓库内提交预编译产物**。

---

## 🚀 3. GitHub Actions 自动化编译流水线

Pi Agent 在仓库 `.github/workflows/build-iso.yml` 路径下写入以下自动化云端压制代码：

```yaml
name: Generate Gateway Warrior ISO

on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Repository
      uses: actions/checkout@v4

    - name: Set up Build Dependencies
      run: |
        sudo apt-get update
        sudo apt-get install -y build-essential libncurses5-dev libssl-dev texinfo \
          unzip help2man bc git wget cpio ccache rsync xorriso locales libelf-dev clang llvm

    - name: Clone Buildroot Toolchain
      run: |
        git clone https://github.com/buildroot/buildroot.git --depth=1 -b 2024.02.x

    - name: Compile xdns-bpf from Source (eBPF bytecode)
      run: |
        cd buildroot
        make defconfig
        # 获取目标内核头文件用于 BPF 编译
        make linux-headers

        # 拉取 xdns-bpf 源码
        git clone https://github.com/liudf0716/xdns-bpf.git --depth=1 /tmp/xdns-bpf
        cd /tmp/xdns-bpf
        # 使用 Buildroot 输出的内核头文件编译 BPF 字节码，保证 ABI 一致
        make KERNEL_HEADERS=$PWD/../../buildroot/output/build/linux-*/usr/include \
          xdns_bpf.o xdns-ctl
        cp xdns_bpf.o xdns-ctl $OLDPWD/
        cd $OLDPWD

    - name: Fetch Pre-built Static Binaries
      run: |
        # xray-lite: 拉取最新 Release
        XL_URL=$(curl -s https://api.github.com/repos/undead-undead/xray-lite/releases/latest \
          | grep "browser_download_url.*linux-amd64" | cut -d '"' -f 4)
        wget -q "${XL_URL}" -O /tmp/xray-lite.tar.gz
        tar -xzf /tmp/xray-lite.tar.gz -C /tmp/
        mv /tmp/xray-lite* buildroot/system/skeleton/usr/bin/xray-lite
        chmod +x buildroot/system/skeleton/usr/bin/xray-lite

        # AdGuardHome: 拉取最新 Release
        AGH_URL=$(curl -s https://api.github.com/repos/AdguardTeam/AdGuardHome/releases/latest \
          | grep "browser_download_url.*linux_amd64.tar.gz" | cut -d '"' -f 4)
        wget -q "${AGH_URL}" -O /tmp/AdGuardHome.tar.gz
        tar -xzf /tmp/AdGuardHome.tar.gz -C /tmp/
        mv /tmp/AdGuardHome/AdGuardHome buildroot/system/skeleton/usr/bin/AdGuardHome
        chmod +x buildroot/system/skeleton/usr/bin/AdGuardHome

    - name: Copy Local eBPF Artifacts
      run: |
        mkdir -p buildroot/system/skeleton/lib/bpf/
        mkdir -p buildroot/system/skeleton/usr/bin/
        cp xdns_bpf.o buildroot/system/skeleton/lib/bpf/
        cp xdns-ctl   buildroot/system/skeleton/usr/bin/
        chmod +x buildroot/system/skeleton/usr/bin/xdns-ctl

    - name: Apply Buildroot Config
      run: |
        cd buildroot
        make defconfig
        # 应用自定义 defconfig (覆盖 ISO 输出、package 选择、内核选项)
        cat ../configs/gateway_defconfig >> .config
        # 补丁式合入内核片段 (BPF/TC/TUN/VirtIO 驱动)
        scripts/kconfig/merge_config.sh .config ../configs/kernel.fragment
        make olddefconfig

    - name: Compile and Press ISO
      run: |
        cd buildroot
        make -j$(nproc)

    - name: Export Ready ISO Artifact
      uses: actions/upload-artifact@v4
      with:
        name: gateway-warrior-iso
        path: buildroot/output/images/rootfs.iso
```

---

## ⚙️ 4. Buildroot 自定义配置文件

### `configs/gateway_defconfig`
追加到默认 `.config`，覆盖镜像类型和 package 选择：

```ini
# 镜像格式：仅输出 ISO
BR2_TARGET_ROOTFS_ISO9660=y
BR2_TARGET_ROOTFS_TAR=n

# 网络核心包
BR2_PACKAGE_IPROUTE2=y
BR2_PACKAGE_IPTABLES=y
BR2_PACKAGE_CURL=y

# QEMU Guest Agent (PVE 摘要页显示 IP)
BR2_PACKAGE_QEMU=y
BR2_PACKAGE_QEMU_GUEST_AGENT=y

# eBPF 依赖
BR2_PACKAGE_LIBELF=y
BR2_PACKAGE_LIBBPF=y
```

### `configs/kernel.fragment`
通过 `merge_config.sh` 合入，启用 eBPF / TC / TUN / VirtIO 必要驱动：

```ini
# === eBPF / TC 过滤器 ===
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_BPF_JIT=y
CONFIG_BPF_JIT_ALWAYS_ON=y
CONFIG_NET_SCH_CLSACT=y
CONFIG_NET_CLS_ACT=y
CONFIG_NET_SCH_INGRESS=y

# === TUN 虚拟网卡 (xray-lite) ===
CONFIG_TUN=y

# === VirtIO 驱动 (PVE 虚拟化) ===
CONFIG_VIRTIO=y
CONFIG_VIRTIO_NET=y
CONFIG_VIRTIO_PCI=y
CONFIG_VIRTIO_BLK=y
CONFIG_DRM_VIRTIO_GPU=n

# === 文件系统 ===
CONFIG_EXT4_FS=y
CONFIG_TMPFS=y

# === 其他 ===
CONFIG_NF_CONNTRACK=y
CONFIG_IP_NF_FILTER=y
CONFIG_IP_NF_NAT=y
```

---

## 🖥️ 5. PVE 9.2.11 虚拟机高级硬件优化参数

将 ISO 上传至 PVE ISO 库，创建 **全虚拟化 VM（非 CT 容器）**，对齐以下参数：

- [ ] **常规 (General)**：勾选"高级"，开启 **Qemu Agent** 开关。
- [ ] **系统 (System)**：机型 **`q35`**，SCSI 控制器 **`VirtIO SCSI single`**。
- [ ] **磁盘 (Disks)**：添加 **`256MB`** 极轻量本地硬盘，总线 **`VirtIO Block`**，开启 **丢弃 (Discard)**。
- [ ] **CPU & 内存**：分配 **1 核 CPU**，内存 **`256MB`** (≥256MB 以覆盖 AdGuard + xray-lite 常驻内存)。
- [ ] **网络 (Network)**：网桥 **`vmbr0`**（单网卡架构），模型 **`VirtIO (半虚拟化)`**，高级设置多队列填 **`1`**，硬件流控与校验和卸载保持开启。

---

## 🤖 6. Pi Agent 自动触发状态机

Pi Agent 接管此工作区时，执行以下步骤：

1. **一键推送**：
   ```bash
   git add . && git commit -m "feat: armed minimal eBPF gateway" && git push origin main
   ```
2. **CI 状态轮询**：通过 GitHub API 轮询 `GET /repos/{owner}/{repo}/actions/runs`，结果为 `success` 时自动将 `gateway-warrior-iso` 产物下载到 `artifacts/` 目录。

---

## 🧪 7. 真机运行状态健康度审计测试指标

虚拟机冷启动后，对以下三维指标进行健康度检查，全绿通过方可对全网下发：

### 📌 测试项 A：系统引导与硬盘自愈
- [ ] PVE 控制台验证虚拟机 **2 秒内** 完成引导至 RAM 运行。
- [ ] PVE 摘要页动态打印网关 VM 的真实局域网 IP，支持网页平滑关机。
- [ ] `df -h` 验证 `/` 为 tmpfs，`/dev/vda` 已挂载到 `/mnt/data`。
- [ ] `ls -l /etc/xray-lite` 验证软链接指向 `/mnt/data/xray-lite`，断电不损坏系统与配置。

### 📌 测试项 B：内核 eBPF 拦截深度观测
- [ ] `tc filter show dev eth0 ingress` 能看到包含 `xdns_bpf.o` 与 `sec ingress` 的内核直接动作特征。
- [ ] `xdns-ctl list-ips` 成功返回内核 BPF Map 结构。

### 📌 测试项 C：联合分流性能指标
修改测试设备的网关与 DNS 指向该 VM IP，分别访问国内与国外目标：

- [ ] **国内流量**：访问 `baidu.com` 秒开，AdGuard 产生去广告日志。`xdns-ctl list-ips` 中 **绝无该国内 IP**（内核层零开销直连）。
- [ ] **国外流量**：访问 `github.com` 秒开，`xdns-ctl list-ips` 中 **该国外 IP 已被内核动态抓取并开始 TTL 倒计时**。
- [ ] **TUN 网卡指纹**：`ip -s link show tun0` 验证 RX/TX 计数器随国外访问产生大量字节流，`xray-lite` 完成安全握手出境。
