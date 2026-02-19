# OpenWrt-CI 项目指南

本仓库基于 [breeze303/openwrt-ci](https://github.com/breeze303/openwrt-ci) 定制，通过 GitHub Actions 云编译 OpenWrt 固件，当前已针对 **ZN M2 (IPQ6000)** 进行优化配置。

## 项目架构

本仓库是 **CI 编排仓库**，不包含 OpenWrt 源码。编译时自动从外部拉取源码：

```
GitHub Actions 触发
    │
    ├── 1. Checkout 本仓库（配置文件 + 脚本）
    │
    ├── 2. Clone OpenWrt 源码（LiBwrt/openwrt-6.x, main-nss 分支）
    │
    ├── 3. 安装 Feeds（官方 + ImmortalWrt 软件源）
    │
    ├── 4. 执行 DIY 脚本（libwrt.sh —— 拉取第三方插件）
    │
    ├── 5. 加载 .config（目标设备 + 软件包选择）
    │
    ├── 6. 编译固件
    │
    └── 7. 上传到 Release
```

## 目录结构

```
openwrt-ci/
├── .github/workflows/          # GitHub Actions 工作流
│   ├── IPQ60XX-6.12-WIFI.yml   # ★ ZN M2 编译工作流（已定制）
│   ├── IPQ60XX-6.12-NOWIFI.yml # IPQ60XX 无 WiFi 版本
│   ├── IPQ60XX-24.10*.yml      # OpenWrt 24.10 分支版本
│   ├── IPQ60XX-ALL.yml         # IPQ60XX 全设备编译
│   ├── IPQ807X-WIFI.yml        # IPQ807X 平台
│   ├── X86-64*.yml             # x86 平台
│   ├── w1701k.yml              # W1701K 设备
│   ├── build-packages.yml      # 单独编译软件包
│   └── Delete-Old-Workflows.yml# 自动清理旧工作流
│
├── configs/                    # 编译配置文件
│   ├── ipq60xx-6.12-wifi.config# ★ ZN M2 专用配置（已定制）
│   ├── ipq60xx-6.12-nowifi.config
│   ├── ipq807x.config
│   ├── x86-64.config
│   ├── x86-64-k6.18.config
│   └── w1701k.config
│
├── scripts/                    # 辅助脚本
│   ├── 011-fix-mbo-modules-build.patch  # hostapd 编译修复补丁
│   ├── init-settings.sh        # 初始化默认设置
│   ├── preset-adguard-core.sh  # AdGuardHome 核心预置
│   ├── preset-clash-core.sh    # Clash 核心预置
│   └── preset-terminal-tools.sh# 终端工具预置
│
├── feeds/                      # Feed 源配置
│   ├── 6.12.txt                # 6.12 内核的 feed 源
│   └── x86-64.txt              # x86 平台的 feed 源
│
├── libwrt.sh                   # ★ ZN M2 DIY 脚本（已定制）
├── diy-script.sh               # 通用 DIY 脚本（完整版，含更多插件）
├── diy-mini.sh                 # 精简 DIY 脚本
├── build.sh                    # 本地编译辅助脚本
├── BUILD-ZN-M2.md              # ZN M2 固件详细文档
└── README.md                   # 原始项目说明
```

> 标记 ★ 的文件是当前已针对 ZN M2 定制的文件。

## 工作流说明

### 当前使用的工作流

**`IPQ60XX-6.12-WIFI.yml`** — ZN M2 专用

| 参数 | 值 |
|------|-----|
| 触发方式 | 手动 (workflow_dispatch) + 定时 (每天 UTC 19:00) |
| OpenWrt 源码 | LiBwrt/openwrt-6.x |
| 分支 | main-nss |
| 配置文件 | configs/ipq60xx-6.12-wifi.config |
| DIY 脚本 | libwrt.sh |
| 工具链缓存 | 启用 |
| 固件发布 | 自动发布到 Release |

### 其他工作流（未定制，保留原版）

| 工作流 | 说明 | 状态 |
|--------|------|------|
| IPQ60XX-6.12-NOWIFI | 无 WiFi 版（做软路由用） | 未定制 |
| IPQ60XX-24.10* | OpenWrt 24.10 稳定分支 | 未定制 |
| IPQ60XX-ALL | 全设备编译（15+ 设备） | 未定制 |
| IPQ807X-WIFI | IPQ807X 平台（如 AX3600） | 未定制 |
| X86-64* | x86 软路由 | 未定制 |
| build-packages | 单独编译软件包 | 未定制 |

## 关键文件详解

### configs/ipq60xx-6.12-wifi.config

编译配置文件，定义了目标设备和所有软件包选择。当前配置：

- **目标**：仅 ZN M2（单设备编译，速度快）
- **内存**：512MB Profile
- **NSS 加速**：启用核心转发加速
- **已选插件**：OpenClash、iStore、UPnP、TTYD、relayd 等
- **已精简**：无 USB、无 Docker、无 DDNS/SQM/WoL/VPN

详细功能清单见 [BUILD-ZN-M2.md](BUILD-ZN-M2.md)。

### libwrt.sh

DIY 脚本，在编译前执行，负责：

1. **修改默认 IP** — `192.168.1.1` → `192.168.20.1`
2. **修复 hostapd** — 应用编译补丁
3. **拉取第三方插件**：
   - OpenClash（dev 分支）
   - iStore 应用商店
   - Argon 主题 + 配置器
   - athena-led（LED 灯控制）
4. **修复 Makefile 路径** — 第三方包的 luci.mk 引用修正
5. **重新安装 feeds** — 确保新插件被索引

### diy-script.sh（参考，未使用）

原版完整 DIY 脚本，包含更多插件（AdGuardHome、MosDNS、SmartDNS、Alist、iStore、NetData 等）。如需更多插件可参考此文件。

## 编译流程

### 方法一：GitHub 网页触发

1. 进入仓库 → **Actions**
2. 左侧选择 **IPQ60XX-6.12-WIFI**
3. 点击 **Run workflow** → **Run workflow**

### 方法二：API 触发

```bash
curl -X POST \
  -H "Authorization: Bearer 你的TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/你的用户名/openwrt-ci/actions/workflows/IPQ60XX-6.12-WIFI.yml/dispatches \
  -d '{"ref":"main"}'
```

返回 HTTP 204 表示触发成功。

### 方法三：定时自动编译

工作流配置了 `cron: 0 19 * * *`（每天北京时间 03:00）自动编译。如不需要，删除 yml 中的 `schedule` 部分：

```yaml
on:
  workflow_dispatch:
  # schedule:              ← 注释掉这两行
  #   - cron: 0 19 * * *
```

## 自定义指南

### 添加新插件

**第一步**：在 `libwrt.sh` 中添加源码拉取

```bash
# 普通克隆
git clone --depth=1 https://github.com/作者/luci-app-xxx package/luci-app-xxx

# 稀疏克隆（仓库很大但只需要一个子目录时使用）
git_sparse_clone 分支 https://github.com/作者/仓库名 目标目录
```

**第二步**：在 `configs/ipq60xx-6.12-wifi.config` 中启用

```
CONFIG_PACKAGE_luci-app-xxx=y
```

### 移除插件

在 config 中注释或改为 `is not set`：

```
# CONFIG_PACKAGE_luci-app-xxx is not set
```

如果 `libwrt.sh` 中有对应的 git clone，也一并删除。

### 修改默认 IP

编辑 `libwrt.sh` 第 7 行：

```bash
sed -i 's/192.168.1.1/你的IP/g' package/base-files/files/bin/config_generate
```

### 修改默认密码

编辑 `libwrt.sh`，添加：

```bash
sed -i '/root/s/x/替换为密码hash/' package/base-files/files/etc/shadow
```

或刷机后在 LuCI → 系统 → 管理权限 中修改。

### 切换到多设备编译

如果需要一次编译多个设备的固件，修改 config 文件头部：

```
CONFIG_TARGET_qualcommax=y
CONFIG_TARGET_qualcommax_ipq60xx=y
CONFIG_TARGET_MULTI_PROFILE=y
CONFIG_TARGET_DEVICE_qualcommax_ipq60xx_DEVICE_zn_m2=y
CONFIG_TARGET_DEVICE_qualcommax_ipq60xx_DEVICE_xiaomi_ax1800=y
CONFIG_TARGET_DEVICE_qualcommax_ipq60xx_DEVICE_redmi_ax5=y
# ... 添加更多设备
```

### 使用其他 DIY 脚本

修改 `.github/workflows/IPQ60XX-6.12-WIFI.yml` 中的 `DIY_SCRIPT`：

```yaml
env:
  DIY_SCRIPT: diy-script.sh   # 使用完整版（更多插件）
  # DIY_SCRIPT: diy-mini.sh   # 使用精简版
  # DIY_SCRIPT: libwrt.sh     # 当前 ZN M2 定制版
```

## 常见问题

### 编译失败怎么办？

1. 进入 Actions → 点击失败的运行记录 → 查看日志
2. 常见原因：
   - **第三方仓库地址变更** — 检查 `libwrt.sh` 中的 git clone 地址
   - **分支名改变** — 如 OpenClash 的 `master` → `dev`
   - **磁盘空间不足** — 减少插件或设备数量
   - **源码编译错误** — 上游代码问题，等待修复

### 为什么 config 中的某些选项编译后没生效？

`make defconfig` 会自动处理依赖关系。如果某个选项依赖的包未选中，该选项会被自动禁用。查看编译产物中的 `build.config` 可以确认最终生效的配置。

### 如何关闭定时自动编译？

编辑 `IPQ60XX-6.12-WIFI.yml`，删除或注释 `schedule` 部分。

### 如何只下载固件不下载全部 Release 文件？

Release 中的关键文件：
- `*-factory.ubi` — Breed 刷机用
- `*-sysupgrade.bin` — 在线升级用
- `build.config` — 实际编译配置（用于排查问题）
- `Packages.tar.gz` — 编译的软件包（iStore/opkg 安装用）

### 工具链缓存是什么？

首次编译需要完整编译工具链（交叉编译器等），非常耗时。`CACHE_TOOLCHAIN: true` 会在首次编译后缓存工具链，后续编译直接复用，大幅缩短编译时间。

## 参考链接

| 资源 | 链接 |
|------|------|
| OpenWrt 源码 (NSS) | https://github.com/LiBwrt/openwrt-6.x |
| 原版 CI 仓库 | https://github.com/breeze303/openwrt-ci |
| NSS 加速包 | https://github.com/qosmio/nss-packages |
| OpenClash | https://github.com/vernesong/OpenClash |
| iStore | https://github.com/linkease/istore |
| Argon 主题 | https://github.com/jerrykuku/luci-theme-argon |
| OpenWrt 官方 | https://openwrt.org |
