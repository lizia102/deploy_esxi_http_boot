# ESXi HTTP Boot Server 自动化部署脚本

## 简介

本脚本用于在 RHEL (Red Hat Enterprise Linux) 系统上自动化部署 VMware ESXi 的 HTTP Boot 网络引导服务器。通过 HTTP Boot 技术，可以实现 ESXi 操作系统的无盘网络安装，适用于大规模服务器部署场景。

## 功能特性

- **全自动部署**：一键完成 HTTP Boot 服务器的所有配置
- **HTTP Boot 支持**：支持 UEFI HTTP Boot 标准，实现真正的 HTTP 网络引导
- **智能文件处理**：自动挂载 ESXi ISO 并提取所有必要文件
- **boot.cfg 自动配置**：自动修改 `prefix=` 配置项，适配 HTTP 路径
- **灵活的服务配置**：可选择是否配置 DHCP 服务器
- **完整的服务栈**：包含 HTTP (Apache)、TFTP、DHCP 服务配置
- **防火墙自动配置**：自动配置 firewalld 规则

## 系统要求

- **操作系统**：RHEL 8/9 或兼容发行版（CentOS Stream, Rocky Linux, AlmaLinux）
- **权限**：需要 root 权限运行
- **网络**：至少一个可用的网络接口
- **存储**：足够的磁盘空间存放 ESXi ISO 和提取的文件（建议 10GB+）
- **软件包**：需要能够访问 RHEL 软件仓库（需注册系统）

## 安装和使用

### 1. 下载脚本

将 `deploy_esxi_http_boot.sh` 脚本上传到 RHEL 服务器。

### 2. 赋予执行权限

```bash
chmod +x deploy_esxi_http_boot.sh
```

### 3. 准备 ESXi ISO

下载 VMware ESXi 安装 ISO 文件，例如：
- VMware ESXi 8.0U2
- VMware ESXi 7.0U3

### 4. 运行脚本

```bash
sudo ./deploy_esxi_http_boot.sh
```

## 执行流程

脚本按照以下逻辑执行：

1. **环境检查**：验证 root 权限，检测网络接口
2. **软件包安装**：安装 httpd, tftp-server, dhcp-server 等必要软件
3. **防火墙配置**：开放 HTTP (80), TFTP (69), DHCP (67) 端口
4. **服务配置**：配置 Apache HTTP 服务器和 TFTP 服务器
5. **ISO 处理**：
   - 挂载 ESXi ISO 文件
   - 拷贝所有文件到 `/var/www/html/esxi/`
   - 卸载 ISO
6. **boot.cfg 配置**（关键步骤）：
   - 读取 `/var/www/html/esxi/efi/boot/boot.cfg`
   - 使用 sed 修改 `prefix=` 行为 `prefix=http://<服务器IP>/esxi`
7. **DHCP 配置**（可选）：
   - 配置 DHCP 服务器指向 HTTP Boot 地址
   - 设置 `vendor-class-identifier` 为 `HTTPClient`
8. **验证和总结**：检查服务状态并输出配置信息

## 配置说明

### boot.cfg 自动修改

脚本会自动处理 `boot.cfg` 文件，这是 ESXi HTTP Boot 的关键配置：

```bash
# 修改前（ISO 原始内容）
prefix=

# 修改后（脚本自动配置）
prefix=http://192.168.1.100/esxi
```

**注意**：对于特定版本的 ESXi（如 8.0U2），建议先手动提取一次 ISO 内的 `boot.cfg`，确认模块列表后再进行批量部署。ESXi 内部路径拼接有时会遇到斜杠问题。

### DHCP 配置

如果选择配置 DHCP 服务器，脚本会生成以下配置：

```dhcp
# HTTP Boot 配置
option vendor-class-identifier "HTTPClient";
option bootfile-name "http://<IP>/esxi/efi/boot/bootx64.efi";

# TFTP 备用配置
filename "bootx64.efi";
next-server <IP>;
```

### 手动配置 DHCP（如果跳过自动配置）

如果使用外部 DHCP 服务器，需要配置以下选项：

- **Option 67 (bootfile-name)**: `http://<服务器IP>/esxi/efi/boot/bootx64.efi`
- **Option 60 (vendor-class-identifier)**: `HTTPClient`

## 客户端配置

### UEFI HTTP Boot 设置

1. 进入目标服务器的 BIOS/UEFI 设置
2. 启用 **HTTP Boot** 或 **UEFI Network Boot**
3. 配置 HTTP Boot URL（某些系统需要手动指定）：
   ```
   http://<服务器IP>/esxi/efi/boot/bootx64.efi
   ```
4. 保存设置并重启

### 网络引导流程

1. 客户端通过 DHCP 获取 IP 地址和引导信息
2. 客户端通过 HTTP 下载 `bootx64.efi` 引导加载程序
3. 引导加载程序读取 `boot.cfg` 配置
4. 通过 HTTP 下载 ESXi 内核和模块
5. 启动 ESXi 安装程序

## 验证和测试

### 检查服务状态

```bash
# 检查 HTTP 服务
systemctl status httpd

# 检查 TFTP 服务
systemctl status tftp

# 检查 DHCP 服务
systemctl status dhcpd
```

### 测试 HTTP 访问

```bash
# 测试 ESXi 文件是否可访问
curl http://<服务器IP>/esxi/

# 检查 boot.cfg 配置
curl http://<服务器IP>/esxi/efi/boot/boot.cfg
```

### 查看日志

```bash
# HTTP 访问日志
tail -f /var/log/httpd/access_log

# HTTP 错误日志
tail -f /var/log/httpd/error_log

# DHCP 日志
tail -f /var/log/messages | grep dhcpd
```

## 故障排除

### boot.cfg 路径问题

如果 ESXi 引导失败，检查 `boot.cfg` 文件：

```bash
cat /var/www/html/esxi/efi/boot/boot.cfg
```

确保 `prefix=` 行正确指向 HTTP URL。

### 模块加载失败

某些 ESXi 版本可能存在路径拼接问题。可以尝试：

1. 手动检查 `boot.cfg` 中的模块路径
2. 根据需要调整路径格式
3. 对于 ESXi 8.0U2+，可能需要移除路径中的多余斜杠

### HTTP Boot 不被识别

确保：
- 客户端支持 UEFI HTTP Boot
- DHCP 配置了正确的 `vendor-class-identifier`
- 防火墙已开放相关端口

### 文件权限问题

确保 Apache 可以读取 ESXi 文件：

```bash
chmod -R 755 /var/www/html/esxi
chown -R apache:apache /var/www/html/esxi
```

## 目录结构

部署完成后，文件结构如下：

```
/var/www/html/esxi/           # ESXi 文件根目录
├── efi/boot/
│   ├── bootx64.efi          # UEFI 引导加载程序
│   └── boot.cfg             # 引导配置文件（已修改）
├── boot.cfg                 # 备用路径
├── b.b00                    # ESXi 内核模块
├── ...                      # 其他 ESXi 文件
└── (其他 ISO 内容)

/var/lib/tftpboot/           # TFTP 根目录（备用）
└── bootx64.efi              # TFTP 引导文件

/etc/dhcp/dhcpd.conf         # DHCP 配置文件
```

## 注意事项

1. **系统注册**：确保 RHEL 系统已注册到 Red Hat，以便安装软件包
2. **网络规划**：如果配置 DHCP，确保 IP 地址范围不与现有网络冲突
3. **ISO 版本**：不同版本的 ESXi 可能有不同的文件结构，建议先测试
4. **安全性**：HTTP Boot 使用明文 HTTP，仅适用于受信任的内部网络
5. **备份**：脚本会备份原始 `boot.cfg` 为 `boot.cfg.bak`

## 示例

### 完整部署示例

```bash
# 1. 准备 ISO 文件
wget https://example.com/VMware-ESXi-8.0U2-22380479.iso

# 2. 运行脚本
sudo ./deploy_esxi_http_boot.sh

# 3. 按提示输入 ISO 路径
# ISO Path: /root/VMware-ESXi-8.0U2-22380479.iso

# 4. 选择是否配置 DHCP
# Configure DHCP server? (y/n): y

# 5. 部署完成，查看输出信息
```

### 仅使用 HTTP 服务（手动配置 DHCP）

```bash
sudo ./deploy_esxi_http_boot.sh
# 在提示时选择不配置 DHCP
# Configure DHCP server? (y/n): n

# 然后手动在现有 DHCP 服务器上配置 Option 67 和 Option 60
```

## 版本历史

- **v1.1** (2026-05-06): 优化执行流程，改进 boot.cfg 处理逻辑
- **v1.0** (2026-05-06): 初始版本

## 许可证

本脚本为开源项目，可自由使用和修改。

## 相关资源

- [VMware ESXi 官方文档](https://docs.vmware.com/en/VMware-vSphere/)
- [UEFI HTTP Boot 规范](https://uefi.org/specs/)
- [Red Hat Enterprise Linux 文档](https://access.redhat.com/documentation/)

## 反馈和问题

如有问题或改进建议，请通过以下方式反馈：
- 提交 Issue 到项目仓库
- 联系系统管理员

---

**免责声明**：本脚本仅供参考和学习使用，请根据实际环境调整配置。在生产环境使用前请充分测试。
