## 概况

* 基于 0.25.1 版本 ```https://github.com/fatedier/frp```

## 最简配置

* 公网机器放 frps 和 frps.ini
* 内网机器放 frpc 和 frpc.ini
* frps.ini

```
# frps.ini
[common]
bind_port = 7000

./frps -c ./frps.ini
```

* frpc.ini

```
# frpc.ini
[common]
server_addr = x.x.x.x
server_port = 7000

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 822

./frpc -c ./frpc.ini
```

* 使用 ```ssh -p 822 root@x.x.x.x```

## 指定域名透传

```
# frps.ini
[common]
bind_port = 7000
vhost_http_port = 8080
```

```
# frpc.ini
[web]
type = http
local_port = 80
custom_domains = www.yourdomain.com
```

## 转发dns查询

```
# frpc.ini
[dns]
type = udp
local_ip = 8.8.8.8
local_port = 53
remote_port = 853
```

```
dig @x.x.x.x -p 853 www.google.com
```

## 转发Unix套接字

```
# frpc.ini
[unix_domain_socket]
type = tcp
remote_port = 860
plugin = unix_domain_socket
plugin_unix_path = /var/run/docker.sock
```

```
curl http://x.x.x.x:860/version
```

## 提供文件服务

```
# frpc.ini
[test_static_file]
type = tcp
remote_port = 888
plugin = static_file
# 要对外暴露的文件目录
plugin_local_path = /tmp/file
# 访问 url 中会被去除的前缀，保留的内容即为要访问的文件路径
plugin_strip_prefix = static
plugin_http_user = abc
plugin_http_passwd = abc
```

* 通过浏览器访问 http://x.x.x.x:888/static/ 来查看位于 /tmp/file 目录下的文件，会要求输入已设置好的用户名和密码

## 安全透传
* 访问者也需要安装frpc，指定访问服务器

```
# frpc.ini
[secret_ssh]
type = stcp
# 只有 sk 一致的用户才能访问到此服务
sk = abcdefg
local_ip = 127.0.0.1
local_port = 22
```

* 在要访问这个服务的机器上启动另外一个frpc，配置如下：

```
# frpc.ini
[common]
server_addr = x.x.x.x
server_port = 7000

[secret_ssh_visitor]
type = stcp
# stcp 的访问者
role = visitor
# 要访问的 stcp 代理的名字
server_name = secret_ssh
sk = abcdefg
# 绑定本地端口用于访问 ssh 服务
bind_addr = 127.0.0.1
bind_port = 822
```

```
ssh -p 822 root@127.0.0.1
```

## 点对点内网穿透
* 双方都要安装frpc

```
# frps.ini
[common]
bind_port = 7000
bind_udp_port = 7001
```

* 准备连接的内网ssh

```
# frpc.ini
[p2p_ssh]
type = xtcp
# 只有 sk 一致的用户才能访问到此服务
sk = abcdefg
local_ip = 127.0.0.1
local_port = 22
```

* 发起连接的机器

```
# frpc.ini
[p2p_ssh_visitor]
type = xtcp
# xtcp 的访问者
role = visitor
# 要访问的 xtcp 代理的名字
server_name = p2p_ssh
sk = abcdefg
# 绑定本地端口用于访问 ssh 服务
bind_addr = 127.0.0.1
bind_port = 822
```

```
ssh -p 822 root@127.0.0.1
```

## 模版变量

```
# frpc.ini
[common]
server_addr = {{ .Envs.FRP_SERVER_ADDR }}
server_port = 7000

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = {{ .Envs.FRP_SSH_REMOTE_PORT }}
```

```
export FRP_SERVER_ADDR="x.x.x.x"
export FRP_SSH_REMOTE_PORT="6000"
./frpc -c ./frpc.ini
```

## 远程管理

### 状态看版 Dashboard

```
# frps.ini
[common]
dashboard_port = 7500
# dashboard 用户名密码，默认都为 admin
dashboard_user = admin
dashboard_pwd = admin
```

### 远程管理(客户端)

```
# frpc.ini
[common]
admin_addr = 127.0.0.1
admin_port = 7400
admin_user = admin
admin_pwd = admin
```

## 身份认证
* 服务端和客户端的 common 配置中的 token 参数一致则身份验证通过。

## 加密和压缩

```
# frpc.ini
[ssh]
type = tcp
local_port = 22
remote_port = 6000
use_encryption = true
use_compression = true  # 压缩算法使用snappy
```

### tls传输

* 通过在 frpc.ini 的 common 中配置 tls_enable = true 来启用此功能，安全性更高。
* 为了端口复用，frp 建立 TLS 连接的第一个字节为 0x17。
* 注意: 启用此功能后除 xtcp 外，不需要再设置 use_encryption。

## 客户端配置更新

* 需要在frpc启用admin管理服务。
* common字段参数无法reload。

```
frpc reload -c ./frpc.ini
```

## 客户端状态

* 需要在frpc启用admin管理服务。

```
frpc status -c ./frpc.ini
```

## 端口白名单

```
# frps.ini
[common]
allow_ports = 2000-3000,3001,3003,4000-50000
```

## 端口复用
* vhost_http_port，vhost_https_port，bind_port可以配置成同一个端口
* 默认开启TCP多路复用，如需关闭，在服务端和客户端的common中分别指定```tcp_mux = false```

## 弱网传输
* 底层通信协议支持选择 kcp 协议，在弱网环境下传输效率提升明显，但是会有一些额外的流量消耗。

```
# frps.ini
[common]
bind_port = 7000
# kcp 绑定的是 udp 端口，可以和 bind_port 一样
kcp_bind_port = 7000

# frpc.ini
[common]
server_addr = x.x.x.x
# server_port 指定为 frps 的 kcp_bind_port
server_port = 7000
protocol = kcp
```

## 连接池
* 默认情况下，当用户请求建立连接后，frps才会请求frpc主动与后端服务建立一个连接。
* 连接池可避免等待与后端服务建立连接以及c/s之间传递控制信息的时间。

```
# frps.ini
[common]
max_pool_count = 5

# frpc.ini
[common]
pool_count = 1
```

## 负载均衡
* 多个相同类型的 proxy 加入到同一个 group 中，从而实现负载均衡的功能。 目前只支持 tcp 类型的 proxy。
* 要求 group_key 相同，做权限验证，且 remote_port 相同。

```
# frpc.ini
[test1]
type = tcp
local_port = 8080
remote_port = 80
group = web
group_key = 123

[test2]
type = tcp
local_port = 8081
remote_port = 80
group = web
group_key = 123
```

## 健康检查
* 在每一个 proxy 的配置下加上 health_check_type = {type} 来启用健康检查功能。
* type 目前可选 tcp 和 http。
* tcp 只要能够建立连接则认为服务正常，http 会发送一个 http 请求，服务需要返回 2xx 的状态码才会被认为正常。

### tcp 示例配置：

```
# frpc.ini
[test1]
type = tcp
local_port = 22
remote_port = 6000
# 启用健康检查，类型为 tcp
health_check_type = tcp
# 建立连接超时时间为 3 秒
health_check_timeout_s = 3
# 连续 3 次检查失败，此 proxy 会被摘除
health_check_max_failed = 3
# 每隔 10 秒进行一次健康检查
health_check_interval_s = 10
```

### http 示例配置：

```
# frpc.ini
[web]
type = http
local_ip = 127.0.0.1
local_port = 80
custom_domains = test.yourdomain.com
# 启用健康检查，类型为 http
health_check_type = http
# 健康检查发送 http 请求的 url，后端服务需要返回 2xx 的 http 状态码
health_check_url = /status
health_check_interval_s = 10
health_check_max_failed = 3
health_check_timeout_s = 3
```

## 修改 Host Header
* 启用host-header修改可以动态修改http请求中的host字段。该功能仅限于http类型的代理。
* http请求中的host字段 test.yourdomain.com 被替换为 dev.yourdomain.com。

```
# frpc.ini
[web]
type = http
local_port = 80
custom_domains = test.yourdomain.com
host_header_rewrite = dev.yourdomain.com
```

## 添加 Http Header
* 对于参数配置中所有以 header_ 开头的参数(支持同时配置多个)，都会被添加到 http 请求的 header 中。

```
# frpc.ini
[web]
type = http
local_port = 80
custom_domains = test.yourdomain.com
header_X-From-Where = frp
```

## 获取用户真实 IP
* 只有 http 类型的代理支持，可以通过用户请求的 header 中的 X-Forwarded-For 和 X-Real-IP 来获取用户真实 IP。

## 配置 HTTP Basic Auth

```
# frpc.ini
[web]
type = http
local_port = 80
custom_domains = test.yourdomain.com
http_user = abc
http_pwd = abc
```

## 自定义二级域名

* 通过在 frps 的配置文件中配置 subdomain_host 启用该特性。
* 在 frpc 的 http、https 代理中可以不配置 custom_domains，而配置一个 subdomain。
* 用户可以通过 subdomain 自行指定自己的 web 服务所需要使用的二级域名，通过 {subdomain}.{subdomain_host} 来访问自己的 web 服务。
* 如果配置 subdomain_host，则 custom_domains 中不能是 subdomain_host 的子域名或者泛域名。
* custom_domains 和 subdomain 可以同时配置。

```
# frps.ini
[common]
subdomain_host = frps.com

# frpc.ini
[web]
type = http
local_port = 80
subdomain = test
```

## URL 路由

```
# frpc.ini
[web01]
type = http
local_port = 80
custom_domains = web.yourdomain.com
locations = /

[web02]
type = http
local_port = 81
custom_domains = web.yourdomain.com
locations = /news,/about
```

## 通过代理连接
* 仅在 protocol = tcp 时生效。

```
# frpc.ini
[common]
server_addr = x.x.x.x
server_port = 7000
http_proxy = http://user:pwd@192.168.1.128:8080
```

## 多端口映射
* 只支持 tcp 和 udp。
* 通过 range: 段落标记来实现，会解析配置将其拆分多个 proxy，每一个proxy以数字后缀命名。

* 例如要映射本地 6000-6005, 6007 这6个端口，主要配置如下：
* 实际连接成功后会创建 8 个 proxy，命名为 test_tcp_0 ～ test_tcp_7。

```
# frpc.ini
[range:test_tcp]
type = tcp
local_ip = 127.0.0.1
local_port = 6000-6006,6007
remote_port = 6000-6006,6007
```

## 插件
* unix_domain_socket、http_proxy、socks5、static_file

### http代理

```
[plugin_http_proxy]
# frpc.ini
type = tcp
remote_port = 804
plugin = http_proxy
plugin_http_user = abc
plugin_http_passwd = abc
```

### socket5代理

```
[plugin_socks5]
# frpc.ini
type = tcp
remote_port = 805
plugin = socks5
plugin_user = abc
plugin_passwd = abc
```

## 源码编译
* 依赖

```
#Ubuntu
apt-get install bison ed gawk gcc libc6-dev make

#CentOS
yum install gcc
```

* go 支持包 ```https://www.golangtc.com/static/go/```

```
wget https://www.golangtc.com/static/go/1.9.2/go1.9.2.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.9.2.linux-amd64.tar.gz

/etc/profile 增加：
export PATH=$PATH:/usr/local/go/bin
export GOPATH=/usr/local/gopath

source /etc/profile
```

* 下载编译

```
go get github.com/fatedier/frp 
cd /usr/local/gopath/src/github.com/fatedier/frp/
make
```

* frp里会多出一个bin目录，放着frpc和frps

* 如：修改 404 页面 ```utils/vhost/resource.go```


## frps 全部配置

```
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
vhost_http_port = 80
vhost_https_port = 443

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

# if subdomain_host is not empty, you can set subdomain when type is http or https in frpc's configure file
# when subdomain is test, the host used by routing is test.frps.com
subdomain_host = frps.com

# if tcp stream multiplexing is used, default is true
tcp_mux = true
```

## frpc 全部配置

```
# [common] is integral section
[common]
# A literal address or host name for IPv6 must be enclosed
# in square brackets, as in "[::1]:80", "[ipv6-host]:http" or "[ipv6-host%zone]:80"
server_addr = 0.0.0.0
server_port = 7000

# if you want to connect frps by http proxy or socks5 proxy, you can set http_proxy here or in global environment variables
# it only works when protocol is tcp
# http_proxy = http://user:passwd@192.168.1.128:8080
# http_proxy = socks5://user:passwd@192.168.1.128:1080

# console or real logFile path like ./frpc.log
log_file = ./frpc.log

# trace, debug, info, warn, error
log_level = info

log_max_days = 3

# for authentication
token = 12345678

# set admin address for control frpc's action by http api such as reload
admin_addr = 127.0.0.1
admin_port = 7400
admin_user = admin
admin_pwd = admin

# connections will be established in advance, default value is zero
pool_count = 5

# if tcp stream multiplexing is used, default is true, it must be same with frps
tcp_mux = true

# your proxy name will be changed to {user}.{proxy}
user = your_name

# decide if exit program when first login failed, otherwise continuous relogin to frps
# default is true
login_fail_exit = true

# communication protocol used to connect to server
# now it supports tcp and kcp and websocket, default is tcp
protocol = tcp

# if tls_enable is true, frpc will connect frps by tls
tls_enable = true

# specify a dns server, so frpc will use this instead of default one
# dns_server = 8.8.8.8

# proxy names you want to start divided by ','
# default is empty, means all proxies
# start = ssh,dns

# heartbeat configure, it's not recommended to modify the default value
# the default value of heartbeat_interval is 10 and heartbeat_timeout is 90
# heartbeat_interval = 30
# heartbeat_timeout = 90

# 'ssh' is the unique proxy name
# if user in [common] section is not empty, it will be changed to {user}.{proxy} such as 'your_name.ssh'
[ssh]
# tcp | udp | http | https | stcp | xtcp, default is tcp
type = tcp
local_ip = 127.0.0.1
local_port = 22
# true or false, if true, messages between frps and frpc will be encrypted, default is false
use_encryption = false
# if true, message will be compressed
use_compression = false
# remote port listen by frps
remote_port = 6001
# frps will load balancing connections for proxies in same group
group = test_group
# group should have same group key
group_key = 123456
# enable health check for the backend service, it support 'tcp' and 'http' now
# frpc will connect local service's port to detect it's healthy status
health_check_type = tcp
# health check connection timeout
health_check_timeout_s = 3
# if continuous failed in 3 times, the proxy will be removed from frps
health_check_max_failed = 3
# every 10 seconds will do a health check
health_check_interval_s = 10

[ssh_random]
type = tcp
local_ip = 127.0.0.1
local_port = 22
# if remote_port is 0, frps will assign a random port for you
remote_port = 0

# if you want to expose multiple ports, add 'range:' prefix to the section name
# frpc will generate multiple proxies such as 'tcp_port_6010', 'tcp_port_6011' and so on.
[range:tcp_port]
type = tcp
local_ip = 127.0.0.1
local_port = 6010-6020,6022,6024-6028
remote_port = 6010-6020,6022,6024-6028
use_encryption = false
use_compression = false

[dns]
type = udp
local_ip = 114.114.114.114
local_port = 53
remote_port = 6002
use_encryption = false
use_compression = false

[range:udp_port]
type = udp
local_ip = 127.0.0.1
local_port = 6010-6020
remote_port = 6010-6020
use_encryption = false
use_compression = false

# Resolve your domain names to [server_addr] so you can use http://web01.yourdomain.com to browse web01 and http://web02.yourdomain.com to browse web02
[web01]
type = http
local_ip = 127.0.0.1
local_port = 80
use_encryption = false
use_compression = true
# http username and password are safety certification for http protocol
# if not set, you can access this custom_domains without certification
http_user = admin
http_pwd = admin
# if domain for frps is frps.com, then you can access [web01] proxy by URL http://test.frps.com
subdomain = web01
custom_domains = web02.yourdomain.com
# locations is only available for http type
locations = /,/pic
host_header_rewrite = example.com
# params with prefix "header_" will be used to update http request headers
header_X-From-Where = frp
health_check_type = http
# frpc will send a GET http request '/status' to local http service
# http service is alive when it return 2xx http response code
health_check_url = /status
health_check_interval_s = 10
health_check_max_failed = 3
health_check_timeout_s = 3

[web02]
type = https
local_ip = 127.0.0.1
local_port = 8000
use_encryption = false
use_compression = false
subdomain = web01
custom_domains = web02.yourdomain.com

[plugin_unix_domain_socket]
type = tcp
remote_port = 6003
# if plugin is defined, local_ip and local_port is useless
# plugin will handle connections got from frps
plugin = unix_domain_socket
# params with prefix "plugin_" that plugin needed
plugin_unix_path = /var/run/docker.sock

[plugin_http_proxy]
type = tcp
remote_port = 6004
plugin = http_proxy
plugin_http_user = abc
plugin_http_passwd = abc

[plugin_socks5]
type = tcp
remote_port = 6005
plugin = socks5
plugin_user = abc
plugin_passwd = abc

[plugin_static_file]
type = tcp
remote_port = 6006
plugin = static_file
plugin_local_path = /var/www/blog
plugin_strip_prefix = static
plugin_http_user = abc
plugin_http_passwd = abc

[secret_tcp]
# If the type is secret tcp, remote_port is useless
# Who want to connect local port should deploy another frpc with stcp proxy and role is visitor
type = stcp
# sk used for authentication for visitors
sk = abcdefg
local_ip = 127.0.0.1
local_port = 22
use_encryption = false
use_compression = false

# user of frpc should be same in both stcp server and stcp visitor
[secret_tcp_visitor]
# frpc role visitor -> frps -> frpc role server
role = visitor
type = stcp
# the server name you want to visitor
server_name = secret_tcp
sk = abcdefg
# connect this address to visitor stcp server
bind_addr = 127.0.0.1
bind_port = 9000
use_encryption = false
use_compression = false

[p2p_tcp]
type = xtcp
sk = abcdefg
local_ip = 127.0.0.1
local_port = 22
use_encryption = false
use_compression = false

[p2p_tcp_visitor]
role = visitor
type = xtcp
server_name = p2p_tcp
sk = abcdefg
bind_addr = 127.0.0.1
bind_port = 9001
use_encryption = false
use_compression = false
```