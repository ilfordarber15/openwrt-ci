# ZN M2 (IPQ6000) OpenWrt 云编译指南

## 设备信息

| 项目 | 参数 |
|------|------|
| 设备型号 | ZN M2 (兆能 M2) |
| CPU | Qualcomm IPQ6000 (ARM Cortex-A53 四核) |
| 内存 | 512MB RAM |
| 闪存 | 128MB NAND Flash |
| 网口 | 1 WAN + 3 LAN (千兆) |
| WiFi | 2.4GHz + 5GHz (802.11ax / Wi-Fi 6) |
| USB | 无 |
| Bootloader | Breed |

## 固件信息

| 项目 | 参数 |
|------|------|
| OpenWrt 源码 | [LiBwrt/openwrt-6.x](https://github.com/LiBwrt/openwrt-6.x) (main-nss 分支) |
| CI 仓库 | [breeze303/openwrt-ci](https://github.com/breeze303/openwrt-ci) (定制版) |
| 内核版本 | Linux 6.12 (主线) |
| 目标平台 | qualcommax / ipq60xx |
| NSS 硬件加速 | 已启用 (固件 v12.2) |
| 内存配置 | 512MB Profile |
| 默认管理地址 | `192.168.20.1` |
| 默认密码 | `password` |
| 主题 | Argon + Bootstrap |
| 语言 | 中文简体 |

## 已安装功能

### 核心功能

| 功能 | 插件 | 说明 |
|------|------|------|
| 科学上网 | OpenClash | Clash 代理 + DNS 分流 |
| 无线中继 | relayd + luci-proto-relay | Client + AP 桥接模式 |
| 应用商店 | iStore | 在线安装/卸载插件 |
| 防火墙 | firewall4 (nftables) | 含 FullCone NAT |
| UPnP | miniupnpd | 端口自动映射 (游戏/P2P) |
| 网页终端 | TTYD | 浏览器内 SSH 终端 |
| 包管理器 | opkg (LuCI 界面) | 管理软件包 |

### 系统特性

| 特性 | 说明 |
|------|------|
| NSS 硬件加速 | 网络转发硬件卸载，接近线速 |
| ZRAM | 内存压缩交换，扩展可用内存 |
| WPA3 | wpad-openssl，支持 WPA2/WPA3 混合加密 |
| IPv6 | 完整 IPv6 支持 |
| FullCone NAT | 全锥形 NAT，游戏/P2P 更友好的 NAT 类型 |
| TUN 支持 | OpenClash TUN 模式透明代理 |

### 未安装（精简掉的）

| 功能 | 原因 |
|------|------|
| AdGuardHome / SmartDNS / MosDNS | 使用 OpenClash 内置 DNS 分流 |
| DDNS | 无公网 IP |
| SQM QoS | 不需要限速 |
| 网络唤醒 (WoL) | 不需要 |
| ZeroTier / WireGuard | 不需要 VPN/远程组网 |
| Docker | 128MB flash 空间不足 |
| USB 驱动 | ZN M2 无 USB 接口 |
| Coremark / 网速测试 | 非必要工具 |
| 隧道协议 (GRE/SIT/6rd/VXLAN) | 家用不需要 |

## 云编译使用方法

### 1. Fork 仓库

将 CI 仓库 Fork 到你的 GitHub 账号。

### 2. 关键文件

| 文件 | 作用 |
|------|------|
| `.github/workflows/IPQ60XX-6.12-WIFI.yml` | GitHub Actions 编译工作流 |
| `configs/ipq60xx-6.12-wifi.config` | 编译配置（目标设备、软件包） |
| `libwrt.sh` | DIY 脚本（拉取第三方插件、修改默认设置） |

### 3. 触发编译

1. 进入 Fork 后的仓库页面
2. 点击 **Actions** 标签页
3. 左侧选择 **IPQ60XX-6.12-WIFI**
4. 点击 **Run workflow** → **Run workflow**

也可通过 GitHub API 触发：

```bash
curl -X POST \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/你的用户名/openwrt-ci/actions/workflows/IPQ60XX-6.12-WIFI.yml/dispatches \
  -d '{"ref":"main"}'
```

### 4. 下载固件

编译完成后：

- **Release**：仓库 → Releases → `IPQ60XX-6.12-WIFI` 标签下载
- **Artifact**：Actions → 对应运行记录 → Artifacts 下载

## 刷机方法

### 首次刷机（通过 Breed）

1. 断电，按住路由器 Reset 键不放，接通电源
2. 约 5 秒后松开，电脑浏览器访问 `192.168.1.1` 进入 Breed
3. 选择 **固件更新**
4. 上传 `factory.ubi` 文件
5. 点击 **上传**，等待刷写完成
6. 重启后访问 `https://192.168.20.1`

### 后续升级（通过 LuCI）

1. 登录 LuCI 管理界面
2. 进入 **系统** → **备份/升级**
3. 上传 `sysupgrade.bin` 文件
4. 勾选 **保留配置**（可选），点击升级

## 无线中继配置

刷机完成后，配置 Client + AP 无线中继模式：

1. 登录 `192.168.20.1`
2. 进入 **网络** → **无线**
3. 点击 **扫描** → 选择上级路由 WiFi → 加入网络
4. 填写密码，接口名称设为 `wwan`
5. 进入 **网络** → **接口** → 添加新接口
6. 协议选择 **Relay bridge**（中继桥接）
7. 中继接口勾选 `lan` 和 `wwan`
8. 保存并应用

## 自定义修改

### 修改管理 IP

编辑 `libwrt.sh`，修改这一行：

```bash
sed -i 's/192.168.1.1/你想要的IP/g' package/base-files/files/bin/config_generate
```

### 添加/移除软件包

编辑 `configs/ipq60xx-6.12-wifi.config`：

```
# 添加
CONFIG_PACKAGE_luci-app-xxx=y

# 移除
# CONFIG_PACKAGE_luci-app-xxx is not set
```

### 添加第三方插件源

编辑 `libwrt.sh`，在 DIY 脚本中添加 git clone：

```bash
git clone --depth=1 https://github.com/xxx/luci-app-xxx package/luci-app-xxx
```

### 升级内存后的修改

如果将 RAM 从 512MB 升级到 1GB，修改 `configs/ipq60xx-6.12-wifi.config`：

```
CONFIG_ATH11K_MEM_PROFILE_1G=y
# CONFIG_ATH11K_MEM_PROFILE_512M is not set
CONFIG_IPQ_MEM_PROFILE_1024=y
# CONFIG_IPQ_MEM_PROFILE_512 is not set
CONFIG_KERNEL_IPQ_MEM_PROFILE=1024
```

注意：更换内存颗粒后还需刷入对应的 CDT（Configuration Data Table）才能正常启动。

## 注意事项

- 128MB NAND 空间有限，不要安装过多软件包
- 编译工具链有缓存机制（`CACHE_TOOLCHAIN: true`），二次编译更快
- 工作流每天 UTC 19:00（北京时间 03:00）自动编译，可在 yml 中关闭 `schedule`
- 如编译失败，查看 Actions 日志定位错误，常见原因：
  - 第三方插件源地址变更或分支改名
  - 磁盘空间不足（插件过多）
  - 上游源码更新导致的兼容性问题
- 刷机有风险，建议保留 Breed 引导器作为恢复手段
