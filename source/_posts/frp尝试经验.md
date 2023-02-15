---
title: frp尝试经验
date: 2023-02-15 16:52:17
tags:
---
# frp使用经验

## Github地址
[github.com/fatedier/frp](https://github.com/fatedier/frp)

## 目录结构
- frps 服务端执行程序
- frpc
- frps.ini 服务端配置文件
- frps_full.ini
- frpc.ini
- frpc_full.ini
- LICENSE

## Server
### 配置文件
frps.ini配置如下
```ini
[common]
# frp监听的端口
bind_port = 7070
# 授权码
token = xxxxxxxxx

# frp管理后台端口
dashboard_port = 7071
# frp管理后台用户名和密码
dashboard_user = admin
dashboard_pwd = xxxxxxxxx
# 复用端口
enable_prometheus = true

# frp日志配置
# log_file = /var/log/frps.log
# log_level = info
# log_max_days = 3
```

### 启动命令
```shell
frps -c frps.ini
```
### 启动服务
frps.service如下
```ini
[Unit]
Description=frpc service
After=network.target syslog.target
Wants=network.target
[Service]
Type=simple
# Restart=always
# Restart=on-failure
# RestartSec=5s
# 启动服务的命令（此处写你的frpc的实际安装目录）
ExecStart=/xxx/xxx/frpc -c /xxx/xxx/frpc.ini
[Install]
WantedBy=multi-user.target
```
可执行文件和配置分别放到/usr/bin /etc/frp

执行命令启动服务
```shell
sudo cp systemd/frps.service /usr/lib/systemd/system/
sudo systemctl enable frps
sudo systemctl start frps
```

查看端口
```shell
netstat -ap | grep 7070
```

## Client
### 配置
frpc.ini如下
```ini
[common]
# 服务端ip
server_addr = 1.117.58.61
# 服务端的bind_port
server_port = 7070

[ssh] # 名称（全局唯一）
type = tcp # 连接类型
local_ip = 127.0.0.1 # 内网地址
local_port = 22 # 内网端口
remote_port = 6000 # 穿透端口
# 当类型为http/https时默认端口为80/443，需要配置自定义域名，无需配置remote_port
# custom_domain = home.cprrr.tech 
```

### 启动
```shell
./frpc.exe -c frpc.ini
```