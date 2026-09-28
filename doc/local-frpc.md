# 本机 frpc 部署

这台 Mac 作为 frp 客户端，把本机 `127.0.0.1:8899` 暴露到公网。

| 项 | 值 |
| --- | --- |
| 系统 | macOS arm64 |
| 客户端 | frpc 0.71.0（Homebrew） |
| 服务端 | `114.132.171.202:7000` |
| 外网访问 | `http://114.132.171.202:8899` |
| 本机服务 | `127.0.0.1:8899` |
| 配置文件 | `/opt/homebrew/etc/frp/frpc.toml` |
| 日志 | `/opt/homebrew/var/log/frpc.log` |
| 开机启动 | `~/Library/LaunchAgents/sh.brew.frpc.plist` |

外网请求到达服务端 `8899` 后，转到这台电脑的 `127.0.0.1:8899`。本机该端口需要有程序在监听，否则外网打开页面会失败。

访问 VPC 时，本机在 `127.0.0.1:1080` 提供 SOCKS5。VPC 内的机器连上同一个 frps 并发布 `vpc-socks` 之后，这条入口才会通。公网 frps 不用为这条隧道另加代理配置。

## 安装

```bash
brew install frpc
```

二进制路径：`/opt/homebrew/opt/frpc/bin/frpc`

## 配置

配置文件：`/opt/homebrew/etc/frp/frpc.toml`

```toml
# frp 客户端配置
serverAddr = "114.132.171.202"
serverPort = 7000

# Web 服务穿透配置
[[proxies]]
name = "web"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8899
remotePort = 8899

# 访问 VPC：连到 VPC 内 frpc 发布的 socks5
[[visitors]]
name = "vpc-socks-visitor"
type = "stcp"
serverName = "vpc-socks"
secretKey = "de3a093717848a30b85c9e9b21411a1d4336ba1989734397"
bindAddr = "127.0.0.1"
bindPort = 1080
```

修改配置后执行 `brew services restart frpc`。

## 访问 VPC

本机入口：`127.0.0.1:1080`，SOCKS5 用户名 `vpc`，密码 `38443cdf0974b5e29f859ceb11523600`。

```bash
curl --socks5-hostname vpc:38443cdf0974b5e29f859ceb11523600@127.0.0.1:1080 http://10.0.1.20:80
```

把下面这份配置放到 VPC 内、能访问内网地址的机器上，然后启动 frpc。`localIP` 不用写，流量从那台机器发出。

```toml
serverAddr = "114.132.171.202"
serverPort = 7000

[[proxies]]
name = "vpc-socks"
type = "stcp"
secretKey = "de3a093717848a30b85c9e9b21411a1d4336ba1989734397"
[proxies.plugin]
type = "socks5"
username = "vpc"
password = "38443cdf0974b5e29f859ceb11523600"
```

## 开机自动启动

用户 `fj` 登录后自动启动 frpc。进程退出后会自动拉起。配置文件是 `/opt/homebrew/etc/frp/frpc.toml`。

启动项：`~/Library/LaunchAgents/sh.brew.frpc.plist`

| 项 | 值 |
| --- | --- |
| Label | `sh.brew.frpc` |
| RunAtLoad | `true` |
| KeepAlive | `true` |
| 运行用户 | `fj` |
| 标准输出 / 错误 | `/opt/homebrew/var/log/frpc.log` |

这是用户登录时加载的 LaunchAgent。未登录时不会运行。

```bash
brew services start frpc
brew services stop frpc
brew services restart frpc
brew services info frpc
launchctl print "gui/$(id -u)/sh.brew.frpc"
```

`brew services stop frpc` 会取消开机启动。需要恢复时再执行 `brew services start frpc`。

前台运行（不走后台服务）：

```bash
/opt/homebrew/opt/frpc/bin/frpc -c /opt/homebrew/etc/frp/frpc.toml
```

## 确认

服务端登录成功时，日志里会出现 `login to server success`、`[web] start proxy success`，以及 `visitor added: [vpc-socks-visitor]`。

```bash
tail -n 40 /opt/homebrew/var/log/frpc.log
lsof -nP -iTCP:1080 -sTCP:LISTEN
lsof -nP -iTCP:8899 -sTCP:LISTEN
```

`127.0.0.1:1080` 在监听只说明本机访问入口已启动。VPC 内的 frpc 未启动时，通过这个端口访问内网会失败。
