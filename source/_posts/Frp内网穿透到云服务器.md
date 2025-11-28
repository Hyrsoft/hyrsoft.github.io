---
title: Frp内网穿透到云服务器
date: 2025-11-28 14:59:32
tags: Frp
categories: 环境配置&工具使用
---
# Frp内网穿透

把本地服务器的端口映射到公网服务器，通过公网服务器ip地址+端口来访问内网

具体为：![截屏2025-11-28 14.11.44](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20251128141147350.png)

其中`192.168.31.29`为本地服务器内网ip

## 公网服务器（Frp服务端）

下载amd64/Linux的frp

```bash
wget https://github.com/fatedier/frp/releases/download/v0.65.0/frp_0.65.0_linux_amd64.tar.gz
```

解压

```bash
tar -zxvf frp_0.65.0_linux_amd64.tar.gz
```

把frpc复制到`/usr/local/bin`

```bash
cd frp_0.65.0_linux_amd64/

sudo cp frps /usr/local/bin/
```

设置配置文件

```bash
sudo mkdir -p /etc/frp

sudo vim /etc/frp/frps.toml
```

填入以下内容

`auth.token`和`webServer.password`要配置的足够安全，不然有安全隐患。

```toml
# /etc/frp/frps.toml

# frp 服务端监听的控制端口，用于和客户端建立连接
bindPort = 7000

# 鉴权机制 
auth.method = "token"
auth.token = "password" 

# FRP Dashboard 面板，方便监控连接数和流量
webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "password"
```

创建和配置`/etc/systemd/system/frps.service`

```toml
[Unit]
Description=Frp Server Service
After=network.target

[Service]
Type=simple
User=root
Restart=on-failure
RestartSec=5s
# 注意配置文件的路径
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml

[Install]
WantedBy=multi-user.target
```

启动和检查状态

```bash
sudo systemctl enable --now frps

systemctl status frps
```

**云服务器放行端口**：

**公网防火墙 (安全组)**： 在服务器的控制台的**安全组** 中放行以下端口：

- `7000` (FRP 通信)
- `8000` (映射后的 WS)
- `8002` (映射后的 Web)
- `8003` (映射后的 Vision)

以及确保云服务器中的对应端口没有被占用

可以通过服务器ip和7500端口进入控制面板

![截屏2025-11-28 13.52.28](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20251128135233845.png)

---

## 内网服务器（Frp客户端）

同样的下载和解压Frp，把frpc复制到`/usr/local/bin`

```bash
sudo cp frpc /usr/local/bin/
```

创建配置文件

```bash
sudo mkdir -p /etc/frp

sudo vim /etc/frp/frpc.toml
```

```toml
# /etc/frp/frpc.toml

# 公网服务器 IP
serverAddr = "x.x.x.x"
serverPort = 7000

# 鉴权 Token (必须与服务端 frps.toml 一致)
auth.method = "token"
auth.token = "password"

# --- 隧道 1: WebSocket API ---
[[proxies]]
name = "xiaozhi-ws-8000"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8000
remotePort = 8000
transport.useEncryption = true
transport.useCompression = true

# --- 隧道 2: Web 管理 & OTA ---
[[proxies]]
name = "xiaozhi-web-8002"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8002
remotePort = 8002
transport.useEncryption = true
transport.useCompression = true

# --- 隧道 3: 视觉分析接口 ---
[[proxies]]
name = "xiaozhi-vision-8003"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8003
remotePort = 8003
transport.useEncryption = true
transport.useCompression = true
```

对应内网服务器的8000、8002、8003端口

![截屏2025-11-28 14.11.44](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20251128141147350.png)

临时测试

```bash
frpc -c frpc.toml
```

参考云服务器配置，设置为systemd守护进程并启动运行

创建和配置`/etc/systemd/system/frpc.service`

```toml
[Unit]
Description=Frp Client Service
After=network.target

[Service]
Type=simple
User=root
Restart=on-failure
RestartSec=5s
# 注意配置文件的路径
ExecStart=/usr/local/bin/frpc -c /etc/frp/frpc.toml

[Install]
WantedBy=multi-user.target
```

启动和检查状态

```bash
sudo systemctl enable --now frpc

systemctl status frpc
```

## 理论上实现的效果

假设公网服务器ip地址为`xxx.xxx.xxx.xxx`，浏览器输入`http://xxx.xxx.xxx.xxx:8002`即可进入AI对话服务端的web界面，其他以此类推。

后面再在公网服务器上部署 Nginx ，把形如`http://xxx.com/xiaozhi/ota/`到地址指向指定端口。
