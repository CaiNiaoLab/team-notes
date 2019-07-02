# 前置需求：
### 1、可访问公网的树莓派（无线或有线连接）
### 2、一个公网IP（我的是阿里云9.9CNY服务器）
# 正文部分
## 一、在拥有公网IP的设备上部署frp的服务端。
##### 1、根据设备型号下载frp服务端
[frp下载地址](https://github.com/fatedier/frp/releases "frp下载地址")。
可以通过 wget 命令下载，例如
`wget https://github.com/fatedier/frp/releases/download/v0.27.0/frp_0.27.0_linux_amd64.tar.gz`
##### 2、解压文件
使用 `tar -zxvf frp_0.27.0_linux_amd64.tar.gz` 进行解压。
然后使用 `cd frp_0.27.0_linux_amd64` 进入目录
目录结构：![](http://47.75.179.208:8000/wp-content/uploads/2019/05/ac8d3cc21ea832273bd452016682c462.png)
##### 3、更改frps.ini内容
这里附上我的配置（#后为注释）：
```shell
# [common] is integral section
[common]
# A literal address or host name for IPv6 must be enclosed
# in square brackets, as in "[::1]:80", "[ipv6-host]:http" or "[ipv6-host%zone]:80"
bind_addr = 0.0.0.0
bind_port = 7000

# udp port to help make udp hole to penetrate nat
bind_udp_port = 7001

# udp port used for kcp protocol, it can be same with 'bind_port'
# if not set, kcp is disabled in frps
kcp_bind_port = 7000

# specify which address proxy will listen for, default value is same with bind_addr
# proxy_bind_addr = 127.0.0.1

# if you want to support virtual host, you must set the http port for listening (optional)
# Note: http port and https port can be same with bind_port
# vhost_http_port = 80
# vhost_https_port = 443

# response header timeout(seconds) for vhost http server, default is 60s
# vhost_http_timeout = 60

# set dashboard_addr and dashboard_port to view dashboard of frps
# dashboard_addr's default value is same with bind_addr
# dashboard is available only if dashboard_port is set
dashboard_addr = 0.0.0.0
dashboard_port = 7500

# dashboard user and passwd for basic auth protect, if not set, both default value is admin
dashboard_user = admin
dashboard_pwd = admin

# dashboard assets directory(only for debug mode)
# assets_dir = ./static
# console or real logFile path like ./frps.log
log_file = ./frps.log

# trace, debug, info, warn, error
log_level = info

log_max_days = 3

# auth token
token = 12345678

# heartbeat configure, it's not recommended to modify the default value
# the default value of heartbeat_timeout is 90
# heartbeat_timeout = 90

# only allow frpc to bind ports you list, if you set nothing, there won't be any limit
allow_ports = 2000-3000,3001,3003,4000-50000

# pool_count in each proxy will change to max_pool_count if they exceed the maximum value
max_pool_count = 5

# max ports can be used for each client, default value is 0 means no limit
max_ports_per_client = 0

# authentication_timeout means the timeout interval (seconds) when the frpc connects frps
# if authentication_timeout is zero, the time is not verified, default is 900s
authentication_timeout = 900

# if subdomain_host is not empty, you can set subdomain when type is http or https in frpc's configure file
# when subdomain is test, the host used by routing is test.frps.com
subdomain_host = frps.com

# if tcp stream multiplexing is used, default is true
tcp_mux = true

```
之后需要使用system daemon对frp服务进行管理。
##### 4、编写frps.service文件，并启动frp服务
frps.service文件内容：
```shell
[Unit]
Description=frps
After = network.target
[Service]
Type=simple
ExecStart= /yourFprsDir/frp_0.27.0_linux_amd64/frps -c /yourFprsDir/frp_0.27.0_linux_amd64/frps.ini
Restart=always
RestartSec=5
[Install]
WantedBy=multi-user.target
```
如果出现权限不足的问题，尝试运行`chmod +x frps`解决。
之后通过 `systemctl start frps` 启动服务。
通过 `systemctl enable frps` 设置开机自启动。
`systemctl stop frps`停止服务。
`systemctl restart frps`重启服务。
`systemctl status frps` 查看服务状态。
如果使用了上面提供的服务端配置并成功启动。通过访问公网IP+端口（7500）看到以下页面，说明服务端启动成功：
![](http://47.75.179.208:8000/wp-content/uploads/2019/05/2dbf42c4d27b045426366b0023746ebd.png)
## 二、在树莓派上开启frp的客户端
##### 1、下载frp客户端并解压
下载：`wget https://github.com/fatedier/frp/releases/download/v0.27.0/frp_0.27.0_linux_arm64.tar.gz`
解压：`tar -zxvf frp_0.27.0_linux_arm64.tar.gz`
##### 2、配置frpc.ini文件
我的配置：
```shell
[common]
server_addr = yourIP
server_port = 7000
token=12345678
[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000
[vnc-reserve]
type = tcp
local_ip = 127.0.0.1
local_port = 5900
remote_port = 5022
[minecraft]
type = tcp
local_ip = 127.0.0.1
local_port = 25565
remote_port = 5028
```
其中除了common，其他\[]内的命名都是是自定义，用来代理多项服务。
common下的token字段的值要与frps.ini内的token值一致。
##### 4、编写frpc.service文件，并启动frp客户端
frpc.service文件内容：
```shell
[Unit]
Description=frpc
After = network.target
[Service]
Type=simple
ExecStart= /yourFprsDir/frp_0.27.0_linux_amd64/frpc -c /yourFprsDir/frp_0.27.0_linux_amd64/frpc.ini
Restart=always
RestartSec=5
[Install]
WantedBy=multi-user.target
```
通过 `systemctl start frpc`启动客户端。
**并在服务器上开启frpc.ini文件中所涉及到的所有服务器端口remote_port（重点）**
启动成功后 通过访问公网IP+端口（7500）看到![](http://47.75.179.208:8000/wp-content/uploads/2019/05/ca335a2688e2020b4512a81de032c5eb.png)
online 代表服务在线，outlined代表离线。

之后就可以通过SSH随时随地访问你的树莓派了。
