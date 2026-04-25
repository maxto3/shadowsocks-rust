# Windows 上使用 sslocal TUN 模式完整指南

## 概述

sslocal 的 TUN 模式在 Windows 上通过 **Wintun** 驱动创建虚拟网卡，将系统所有流量（TCP/UDP）透明地路由到 shadowsocks 代理服务器。无需逐个配置应用程序的代理设置。

## 前置条件

- shadowsocks-rust 已编译或下载（需启用 `local-tun` feature）
- 管理员权限（创建 TUN 设备需要）
- 网络连接

## 第一步：获取 Wintun 驱动

Wintun 是 WireGuard 项目提供的 TUN 驱动，`tun` crate 在 Windows 上依赖它。

1. 访问 [Wintun 下载页](https://www.wintun.net/)
2. 下载最新版本的 `wintun.dll`
3. 将 `wintun.dll` 放置到以下任一位置：
   - **与 `sslocal.exe` 同目录**
   - **系统 PATH 环境变量中的任意目录**（如 `C:\Windows\System32\`）

> **注意**：如果缺少 `wintun.dll`，sslocal 启动时会报错，无法创建 TUN 设备。

## 第二步：确定网络接口名称

`--outbound-bind-interface` 参数需要指定**物理网卡名称**，用于指定 sslocal 的出站流量走哪张网卡。

在 PowerShell 中运行：

```powershell
Get-NetAdapter | Select-Object Name, InterfaceDescription, Status
```

示例输出：

```
Name                      InterfaceDescription                    Status
----                      --------------------                    ------
Ethernet 0                Realtek PCIe GbE Family Controller      Up
Wi-Fi                     Intel Wi-Fi 6 AX201                     Up
```

根据你的实际网络连接，选择状态为 `Up` 的网卡名称，例如 `"Ethernet 0"` 或 `"Wi-Fi"`。

## 第三步：准备 shadowsocks-rust 二进制文件

### 方式 A：从 Release 下载（推荐）

1. 前往 [shadowsocks-rust Releases](https://github.com/shadowsocks/shadowsocks-rust/releases)
2. 下载 `shadowsocks-vX.X.X-stable.x86_64-pc-windows-msvc.zip`
3. 解压到本地目录，例如 `C:\shadowsocks\`

### 方式 B：自行编译

```powershell
cargo build --release --features "local-tun"
```

编译产物在 `target\release\sslocal.exe`。

## 第四步：启动 sslocal TUN 模式

### 方式 1：命令行参数

以**管理员身份**打开 PowerShell，执行：

```powershell
cd C:\shadowsocks\
.\sslocal.exe --protocol tun `
    -s "your-server-ip:8388" `
    -m "aes-256-gcm" `
    -k "your-password" `
    --outbound-bind-interface "Ethernet 0" `
    --tun-interface-name "shadowsocks"
```

参数说明：

| 参数 | 说明 |
|------|------|
| `--protocol tun` | 启用 TUN 模式 |
| `-s` | shadowsocks 服务器地址和端口 |
| `-m` | 加密方法 |
| `-k` | 密码 |
| `--outbound-bind-interface` | **必填**。物理网卡名称，用于出站流量 |
| `--tun-interface-name` | 可选。TUN 设备名称，默认为 `"tun0"` |

### 方式 2：配置文件

创建 `config.json`：

```jsonc
{
    "locals": [
        {
            "protocol": "tun",
            "tun_interface_name": "shadowsocks",
            "tun_interface_address": "10.255.0.1/24"
        }
    ],
    "server": "your-server-ip",
    "server_port": 8388,
    "method": "aes-256-gcm",
    "password": "your-password",
    "outbound_bind_interface": "Ethernet 0"
}
```

然后以管理员身份运行：

```powershell
.\sslocal.exe -c config.json
```

### 方式 3：使用 SIP008 在线配置 URL

```powershell
.\sslocal.exe --protocol tun `
    --server-url "ss://YOUR-SS-URL" `
    --outbound-bind-interface "Ethernet 0" `
    --tun-interface-name "shadowsocks"
```

## 第五步（可选）：配置 ACL（访问控制列表）

ACL 文件用于控制哪些流量走代理、哪些流量直连（绕过代理）。TUN 模式完全支持 ACL。

### ACL 文件格式

创建 `bypass.acl` 文件：

```
# 注释行以 # 开头

# 直连（绕过代理）的 IP 段
[bypass_list]
# 内网地址段
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
# 本地回环
127.0.0.0/8
# 链路本地
169.254.0.0/16

# 走代理的 IP 段
[proxy_list]
# 默认所有其他流量走代理

# 直连的域名
[bypass_list]
.google.com  # 所有 .google.com 子域名直连
.local

# 走代理的域名
[proxy_list]
# 默认所有其他域名走代理
```

### 使用 ACL

#### 命令行方式

```powershell
.\sslocal.exe --protocol tun `
    -s "your-server-ip:8388" `
    -m "aes-256-gcm" `
    -k "your-password" `
    --outbound-bind-interface "Ethernet 0" `
    --tun-interface-name "shadowsocks" `
    --acl "C:\path\to\bypass.acl"
```

#### 配置文件方式

```jsonc
{
    "locals": [
        {
            "protocol": "tun",
            "tun_interface_name": "shadowsocks",
            "tun_interface_address": "10.255.0.1/24"
        }
    ],
    "server": "your-server-ip",
    "server_port": 8388,
    "method": "aes-256-gcm",
    "password": "your-password",
    "outbound_bind_interface": "Ethernet 0",
    "acl": "C:\\path\\to\\bypass.acl"
}
```

### ACL 工作原理

ACL 在 `ServiceContext` 级别生效，TUN 模式的 TCP 和 UDP 流量都会经过 ACL 检查：

- **TCP 流量**：在 `AutoProxyClientStream::connect_with_opts()` 中调用 `context.check_target_bypassed(&addr)`，根据 ACL 规则决定是走代理还是直连
- **UDP 流量**：在 `UdpAssociationManager::send_to()` 中同样调用 `context.check_target_bypassed()` 进行 ACL 判断

> **注意**：ACL 是全局配置，对所有 local 协议（SOCKS5、TUN、HTTP 等）均有效。也可以在配置文件中为每个 local 实例单独设置私有 ACL。

## 第六步：验证 TUN 模式是否正常工作

### 检查 TUN 设备

在 PowerShell 中查看新增的网卡：

```powershell
Get-NetAdapter | Where-Object Name -eq "shadowsocks"
```

如果成功，会显示一个状态为 `Up` 的虚拟网卡。

### 检查 IP 连通性

```powershell
ipconfig
```

应该能看到名为 `shadowsocks` 的网卡，IP 地址为 `10.255.0.1`（或你在配置中指定的地址）。

### 检查流量是否经过代理

访问 https://ipinfo.io/ 或 https://whatismyip.com/，查看显示的 IP 是否为 shadowsocks 服务器的出口 IP。

## 停止 TUN 模式

按 `Ctrl+C` 停止 sslocal 进程。TUN 设备会自动销毁。

如果 TUN 设备未被清理，可以手动删除：

```powershell
Get-NetAdapter -Name "shadowsocks" | Remove-NetAdapter -Confirm:$false
```

## 常见问题

### Q1: 启动报错 "failed to create tun device"

**原因**：缺少 `wintun.dll` 或未以管理员身份运行。

**解决**：
1. 确认 `wintun.dll` 已放置在正确位置
2. 以**管理员身份**运行 PowerShell/cmd

### Q2: 启动报错 "failed to bind outbound interface"

**原因**：`--outbound-bind-interface` 指定的网卡名称不正确。

**解决**：
1. 运行 `Get-NetAdapter` 确认网卡名称
2. 注意名称中的空格和大小写，例如 `"Ethernet 0"` 而非 `"Ethernet0"`

### Q3: TUN 模式启动成功但无法上网

**可能原因**：
1. 未设置默认路由指向 TUN 设备（sslocal 会自动处理路由，但某些情况下可能需要手动配置）
2. DNS 解析问题

**排查**：
```powershell
# 查看路由表
route print

# 测试 DNS 解析
nslookup google.com

# 测试连通性
ping 8.8.8.8
```

### Q4: 如何同时代理 TCP 和 UDP？

默认 TUN 模式同时代理 TCP 和 UDP。如果需要仅代理 TCP：

```powershell
.\sslocal.exe --protocol tun --mode tcp_only `
    -s "..." -m "..." -k "..." `
    --outbound-bind-interface "Ethernet 0"
```

### Q5: 如何配置多个 shadowsocks 服务器？

在配置文件中使用 `servers` 数组：

```jsonc
{
    "locals": [
        {
            "protocol": "tun",
            "tun_interface_name": "shadowsocks",
            "tun_interface_address": "10.255.0.1/24"
        }
    ],
    "servers": [
        {
            "address": "server1.example.com",
            "port": 8388,
            "method": "aes-256-gcm",
            "password": "password1"
        },
        {
            "address": "server2.example.com",
            "port": 8388,
            "method": "chacha20-ietf-poly1305",
            "password": "password2"
        }
    ],
    "outbound_bind_interface": "Ethernet 0"
}
```

sslocal 会自动进行负载均衡。

## 完整示例

以下是一个完整的端到端示例：

```powershell
# 1. 下载 wintun.dll 并放到 sslocal.exe 同目录

# 2. 以管理员身份打开 PowerShell

# 3. 启动 sslocal TUN 模式
cd C:\shadowsocks\
.\sslocal.exe --protocol tun `
    -s "192.168.1.100:8388" `
    -m "aes-256-gcm" `
    -k "MySecretPassword123" `
    --outbound-bind-interface "Wi-Fi" `
    --tun-interface-name "shadowsocks"

# 4. 验证
#    - 打开浏览器访问 https://ipinfo.io/
#    - 确认 IP 已变为代理服务器的出口 IP
#    - 所有系统流量（浏览器、命令行、应用等）均通过代理

# 5. 停止
#    - 按 Ctrl+C 停止 sslocal
```

## 技术原理

<svg width="720" height="480" xmlns="http://www.w3.org/2000/svg" font-family="Segoe UI, Arial, sans-serif">
  <defs>
    <marker id="arrowBlue" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#1976d2"/>
    </marker>
    <marker id="arrowGreen" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#388e3c"/>
    </marker>
    <marker id="arrowOrange" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#e65100"/>
    </marker>
    <marker id="arrowRed" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#c62828"/>
    </marker>
    <marker id="arrowPurple" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#6a1b9a"/>
    </marker>
    <filter id="shadow" x="-5%" y="-5%" width="115%" height="115%">
      <feDropShadow dx="2" dy="2" stdDeviation="3" flood-opacity="0.15"/>
    </filter>
  </defs>

  <!-- Background -->
  <rect x="0" y="0" width="720" height="480" fill="#fafafa" rx="10"/>

  <!-- Title -->
  <text x="360" y="30" text-anchor="middle" font-size="16" font-weight="bold" fill="#333">sslocal TUN 模式流量路径</text>

  <!-- Windows system boundary -->
  <rect x="20" y="50" width="460" height="380" fill="#e3f2fd" stroke="#90caf9" stroke-width="2" rx="8" stroke-dasharray="6,3"/>
  <text x="250" y="72" text-anchor="middle" font-size="13" font-weight="bold" fill="#1565c0">Windows 系统</text>

  <!-- App -->
  <rect x="60" y="100" width="160" height="60" fill="#e1f5fe" stroke="#4fc3f7" stroke-width="2" rx="6" filter="url(#shadow)"/>
  <text x="140" y="125" text-anchor="middle" font-size="13" font-weight="bold" fill="#0277bd">应用</text>
  <text x="140" y="145" text-anchor="middle" font-size="11" fill="#0277bd">(浏览器 / 命令行 / ...)</text>

  <!-- Route table -->
  <rect x="60" y="200" width="160" height="60" fill="#f3e5f5" stroke="#ce93d8" stroke-width="2" rx="6" filter="url(#shadow)"/>
  <text x="140" y="225" text-anchor="middle" font-size="13" font-weight="bold" fill="#6a1b9a">系统路由表</text>
  <text x="140" y="245" text-anchor="middle" font-size="11" fill="#6a1b9a">指向 TUN 设备</text>

  <!-- TUN device -->
  <rect x="60" y="300" width="160" height="60" fill="#fff3e0" stroke="#ffb74d" stroke-width="2" rx="6" filter="url(#shadow)"/>
  <text x="140" y="325" text-anchor="middle" font-size="13" font-weight="bold" fill="#e65100">TUN 虚拟网卡</text>
  <text x="140" y="345" text-anchor="middle" font-size="11" fill="#e65100">← 读取 IP 包 / 写回响应 →</text>

  <!-- sslocal -->
  <rect x="300" y="200" width="160" height="60" fill="#e8f5e9" stroke="#81c784" stroke-width="2" rx="6" filter="url(#shadow)"/>
  <text x="380" y="225" text-anchor="middle" font-size="13" font-weight="bold" fill="#2e7d32">sslocal (TUN)</text>
  <text x="380" y="245" text-anchor="middle" font-size="11" fill="#2e7d32">解析 TCP/UDP · Shadowsocks 加密</text>

  <!-- Physical NIC -->
  <rect x="300" y="300" width="160" height="60" fill="#fce4ec" stroke="#e57373" stroke-width="2" rx="6" filter="url(#shadow)"/>
  <text x="380" y="325" text-anchor="middle" font-size="13" font-weight="bold" fill="#c62828">物理网卡</text>
  <text x="380" y="345" text-anchor="middle" font-size="11" fill="#c62828">出站流量出口</text>

  <!-- Remote server -->
  <rect x="530" y="200" width="160" height="60" fill="#fce4ec" stroke="#ef5350" stroke-width="2" rx="6" filter="url(#shadow)"/>
  <text x="610" y="225" text-anchor="middle" font-size="13" font-weight="bold" fill="#b71c1c">远端 ssserver</text>
  <text x="610" y="245" text-anchor="middle" font-size="11" fill="#b71c1c">解密 → 访问目标</text>

  <!-- Arrows -->
  <!-- App -> Route -->
  <line x1="140" y1="160" x2="140" y2="195" stroke="#1976d2" stroke-width="2" marker-end="url(#arrowBlue)"/>
  <text x="155" y="182" font-size="10" fill="#1976d2">流量</text>

  <!-- Route -> TUN -->
  <line x1="140" y1="260" x2="140" y2="295" stroke="#1976d2" stroke-width="2" marker-end="url(#arrowBlue)"/>
  <text x="155" y="282" font-size="10" fill="#1976d2">默认路由</text>

  <!-- TUN -> sslocal -->
  <line x1="220" y1="320" x2="295" y2="240" stroke="#e65100" stroke-width="2" marker-end="url(#arrowOrange)"/>
  <text x="235" y="275" font-size="10" fill="#e65100">原始 IP 包</text>

  <!-- sslocal -> Physical NIC -->
  <line x1="380" y1="260" x2="380" y2="295" stroke="#388e3c" stroke-width="2" marker-end="url(#arrowGreen)"/>
  <text x="395" y="282" font-size="10" fill="#388e3c">加密流量</text>

  <!-- Physical NIC -> Remote -->
  <line x1="460" y1="320" x2="525" y2="240" stroke="#c62828" stroke-width="2" marker-end="url(#arrowRed)"/>
  <text x="475" y="275" font-size="10" fill="#c62828">Internet</text>

  <!-- Remote -> Physical NIC (response) -->
  <line x1="610" y1="260" x2="610" y2="340" stroke="#6a1b9a" stroke-width="2" marker-end="url(#arrowPurple)"/>
  <line x1="610" y1="340" x2="460" y2="340" stroke="#6a1b9a" stroke-width="2"/>
  <line x1="460" y1="340" x2="460" y2="325" stroke="#6a1b9a" stroke-width="2" marker-end="url(#arrowPurple)"/>
  <text x="535" y="335" font-size="10" fill="#6a1b9a">响应</text>

  <!-- sslocal -> TUN (response) -->
  <line x1="300" y1="230" x2="225" y2="310" stroke="#6a1b9a" stroke-width="2" marker-end="url(#arrowPurple)"/>
  <text x="240" y="265" font-size="10" fill="#6a1b9a">响应 IP 包</text>

  <!-- TUN -> App (response) - route along bottom and left edges, outside all boxes -->
  <line x1="140" y1="360" x2="140" y2="390" stroke="#6a1b9a" stroke-width="2"/>
  <line x1="140" y1="390" x2="30" y2="390" stroke="#6a1b9a" stroke-width="2"/>
  <line x1="30" y1="390" x2="30" y2="90" stroke="#6a1b9a" stroke-width="2"/>
  <line x1="30" y1="90" x2="140" y2="90" stroke="#6a1b9a" stroke-width="2"/>
  <line x1="140" y1="90" x2="140" y2="100" stroke="#6a1b9a" stroke-width="2" marker-end="url(#arrowPurple)"/>
  <text x="85" y="395" font-size="10" fill="#6a1b9a">响应</text>

  <!-- Legend -->
  <rect x="20" y="440" width="680" height="30" fill="none"/>
  <line x1="30" y1="455" x2="60" y2="455" stroke="#1976d2" stroke-width="2"/>
  <text x="65" y="459" font-size="10" fill="#555">请求路径</text>
  <line x1="160" y1="455" x2="190" y2="455" stroke="#6a1b9a" stroke-width="2"/>
  <text x="195" y="459" font-size="10" fill="#555">响应路径</text>
</svg>

- sslocal 创建 TUN 虚拟网卡，分配 IP 地址（如 `10.255.0.1/24`）
- 系统路由表将默认流量指向 TUN 网卡
- sslocal 从 TUN 设备读取原始 IP 包，解析 TCP/UDP
- 通过 shadowsocks 协议加密后，经物理网卡发往远端服务器
- 远端服务器解密后访问目标，响应原路返回

## 参考

- [shadowsocks-rust README](https://github.com/shadowsocks/shadowsocks-rust#tun-interface-client)
- [Wintun](https://www.wintun.net/) - TUN 驱动
- [shadowsocks-rust Releases](https://github.com/shadowsocks/shadowsocks-rust/releases)