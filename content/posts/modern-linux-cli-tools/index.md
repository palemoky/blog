---
title: Linux 现代终端工具一览
date: 2026-06-25T19:45:23+08:00
draft: false
tags:
  - Linux
---

> 💡 **核心哲学**：为了极大地降低命令行的记忆碎片，建议将精力投资在**正则表达式**与**模糊匹配（如 `fzf`）** 的通用语法上。掌握了这一套搜索逻辑，即可借助现代工具（`fd` / `rg` / `fzf`）通杀涵盖文件寻址、文本过滤、进程检索、甚至历史命令在内的 90% 日常场景，真正做到“一招吃遍天”。

## 命令行工具速查索引

| 类别           | 传统命令               | 现代替代 / 推荐工具                       | 核心用途 / 链接跳转                                           |
| :------------- | :--------------------- | :---------------------------------------- | :------------------------------------------------------------ |
| **文件查找**   | `find`                 | `fd` / `fzf`                              | [文件递归寻找、模糊搜索与交互式补全](#查找命令)               |
| **文件查看**   | `cat` / `ls`           | `bat` / `eza` / `yazi`                    | [带语法高亮查看、树形彩色列表、TUI 文件管理](#文件查看)       |
| **文本过滤**   | `grep` / `diff`        | `rg` / `delta`                            | [高速正则匹配、高亮代码差异比对](#匹配与搜索)                 |
| **文本处理**   | `cut` / `sed` / `awk`  | `jq` / `xargs`                            | [结构化文本/JSON 解析、流编辑与参数转换](#提取与切分)         |
| **网络配置**   | `ifconfig` / `netstat` | `ip` / `ss` / `nmcli`                     | [查看 IP 路由、端口监听、终端连接 Wi-Fi](#状态与配置)         |
| **连通路由**   | `ping` / `traceroute`  | `mtr` / `ping.pe` / `cip.cc`              | [网络链路测试、GFW 封锁排查、IP 归属查询](#连通性与路由)      |
| **端口探测**   | `telnet`               | `nmap` / `nc`                             | [批量扫描开放端口、TCP/UDP 连通性测试与监听](#端口探测与扫描) |
| **文件传输**   | `scp` / `ftp`          | `curl` / `httpie` / `wget` / `rsync`      | [API 测试、断点下载、主机间增量备份同步](#http-通信与传输)    |
| **系统监控**   | `top` / `ps`           | `btop` / `procs` / `systemctl` / `sysctl` | [可视化系统大盘、进程检索、系统服务/内核调优](#进程与系统)    |
| **磁盘管理**   | `du` / `df`            | `dust` / `duf` / `ncdu`                   | [条形图磁盘空间分析、交互式目录清理](#磁盘管理)               |
| **效率辅助**   | `man` / `cd`           | `tldr` / `zoxide` / `zellij`              | [命令行示例速查、智能目录跳转、终端多路复用](#辅助工具)       |
| **安全防火墙** | `iptables`             | `nft` / `ufw`                             | [流量安全过滤、网络端口黑白名单与 Docker 防火墙避坑](#防火墙) |

## Linux 文件目录结构

```text
.
├── bin                 // binary 的缩写，普通用户的可执行命令
├── boot                // 系统启动所需文件
├── dev                 // 设备文件存放目录，体现 Linux “一切皆文件”的思想
│   ├── block
│   ├── disk
│   ├── fd
│   ├── net
│   ├── null
│   ├── stderr
│   ├── stdin
│   └── stdout
├── etc                 // 配置文件保存位置
│   ├── apt
│   ├── ca-certificates // TLS 证书存放路径
│   ├── calendar
│   ├── crontab
│   ├── dhcp
│   ├── php
│   ├── python3
│   ├── resolv.conf     // DNS 配置文件
│   ├── ssh             // SSH 配置路径，修改端口号、关闭密码登陆
│   ├── systemd
│   └── vim
├── home                // 普通用户的主目录
├── lib                 // 执行 bin 或 sbin 所依赖的文件
├── lost+found          // 当系统意外崩溃或意外关机时，产生的一些文件碎片会存放在这里。在系统启动的过程中，fsck 工具会检查这里，并修复已经损坏的文件系统。这个目录只在每个分区中出现，例如，/lost+found 就是根分区的备份恢复目录，/boot/lost+found 就是 /boot 分区的备份恢复目录
├── media               // 挂载媒体设备，如软盘和光盘
├── mnt                 // 挂载U盘、移动硬盘等
├── opt                 // 第三方安装的软件保存位置
├── proc                // 虚拟文件系统。该目录中的数据并不保存在硬盘上，而是保存到内存中，许多命令的输出都是从这个目录中获取的，如 `free`、`uptime`、`lscpu`、`top`、`ps`、`vmstat`、`netstat` 等
├── root                // root 的主目录
├── run                 // 当前正在运行程序的文件
├── sbin                // 系统管理员的可执行命令
├── snap                // 用于存储 snap 包
├── srv                 // 服务数据目录。一些系统服务启动之后，可以在这个目录中保存所需的数据
├── sys                 // 虚拟文件系统。和 /proc/ 目录相似，主要保存与内核相关的信息，数据同样保存于内存中
├── tmp                 // 临时目录，请勿在此处存放重要数据
├── usr                 // Unix Software Resource，用于存储系统软件资源
│   ├── bin             // 存放系统命令，普通用户和root均可执行
│   ├── include         // C/C++ 等编程语言头文件的放置目录
│   ├── lib             // 应用程序调用的函数库保存位置
│   ├── local           // 手动安装的软件保存位置。建议源码包软件安装在此处
│   │   ├── bin
│   │   ├── etc
│   │   ├── games
│   │   ├── include
│   │   ├── lib
│   │   ├── man
│   │   ├── sbin
│   │   ├── share
│   │   └── src
│   ├── sbin            // 存放根文件系统不必要的系统管理命令，如多数服务程序，仅 root 可用
│   ├── share           // 应用程序的资源文件保存位置，如帮助文档、说明文档和字体目录
│   └── src             // 源码包保存位置，用户下载的源码包推荐放在 /usr/local/src/，内核源码放到 /usr/src/linux/ 中
└── var                 // 用于存储动态数据，例如缓存、日志文件、软件运行过程中产生的文件等
    ├── backups
    ├── cache
    ├── lib
    ├── local
    ├── lock
    ├── log             // 系统和应用程序的日志文件放置目录
    ├── mail
    ├── opt
    ├── run
    ├── snap
    ├── spool          // 一些临时存放，随时会被用户所调用的数据，如 mail 和 cron
    ├── swap
    ├── tmp
    └── www
```

## 查找命令

### find

按条件递归查找文件和目录。

```bash
find /path -name "*.log"                 # 按名称查找
find /path -type f -name "*.conf"        # 按类型查找（f 文件，d 目录，l 符号链接）
find /path -type f -mtime -7             # 按修改时间查找（7 天内修改过的文件）
find /path -type f -size +100M           # 按文件大小查找（大于 100M 的文件）
find /path -name "*.tmp" -exec rm {} \;  # 查找并执行操作（删除所有 .tmp 文件）
find /path -empty                        # 查找空文件或空目录
find /path -perm 755                     # 按权限查找
```

> 💡 现代替代：[fd](https://github.com/sharkdp/fd) — 语法更简洁，默认忽略 `.gitignore`，彩色输出。

```bash
fd "\.log$"                   # 按正则查找
fd -t f "readme"              # 只查找文件
fd -e md                      # 按扩展名查找
fd -E node_modules "\.js$"    # 排除目录
fd -e log -x rm {}            # 查找并执行
```

### fzf

[fzf](https://github.com/junegunn/fzf) 是一个宇宙级的通用终端模糊查找器。它不仅可以作为管道过滤器，本身更是命令补全和快捷跳转的外挂神器。

#### 1. 基础管道与组合

```bash
fzf                                         # 独立运行：直接模糊搜索当前目录下所有文件
fzf --preview 'cat {}'                      # 预览文件内容

vim $(fzf)                                  # 神奇组合：先搜索，选中后直接用 vim 打开
kill -9 $(ps aux | fzf | awk '{print $2}')  # 交互式拔管：全屏检索进程，回车直接 kill
git branch | fzf | xargs git checkout       # 交互式切分支
```

#### 2. 快捷键触发（极力推荐配置）

安装 `fzf` 时如果写入了 shell 配置文件，将解锁三大灵魂快捷键：

- **`Ctrl + R`**：**历史命令搜索**（彻底取代原生又丑又难用的 `reverse-i-search`，大屏模糊匹配旧命令）。
- **`Ctrl + T`**：**插入路径**（在编写任何命令时按下，立刻搜索当前目录文件，回车后将路径自动贴入光标处）。
- **`Alt/Esc + C`**：**神级跳转**（按下立即搜索所有子目录，选中即可 `cd` 进入目标文件夹）。

#### 3. Shell 终极补全 (`**<TAB>`)

在任意需要填入路径 / 进程号 / 主机的命令后输入 `**` 并按下 `Tab`，可以直接触发全屏选择器供你“填空”：

```bash
vim **<TAB>               # 模糊搜索并打开深层文件
kill -9 **<TAB>           # 模糊搜索并填入进程 PID
ssh **<TAB>               # 模糊挑选 ~/.ssh/config 里的主机名并连接
```

## 网络

### 状态与配置

#### ip

`ip` 命令用于管理网络接口、路由和地址，是 `ifconfig`、`arp` 和 `route` 的替代品。

```bash
ip addr show                           # 查看所有网络接口及 IP
ip a                                   # 简写

ip addr show eth0                      # 查看指定接口

ip addr add 192.168.1.100/24 dev eth0  # 添加/删除 IP 地址
ip addr del 192.168.1.100/24 dev eth0

ip link set eth0 up                    # 启用/禁用接口
ip link set eth0 down

ip route show                          # 查看路由表
ip r                                   # 简写

ip route add default via 192.168.1.1   # 添加默认网关

ip neigh show                          # 查看邻居表（ARP）
```

#### ss

`ss`（Socket Statistics）用于查看网络连接、端口监听等信息。是 `netstat` 的替代品。

```bash
ss -t                         # 查看所有 TCP 连接
ss -lntp                      # 查看所有监听端口（非root时隐藏其他用户的进程）
ss -lntp | grep :80           # 查看指定端口的连接
ss -lnup                      # 查看所有 UDP 端口
ss state established          # 按状态过滤
ss -s                         # 统计各状态的连接数
```

#### nmcli

`nmcli` 是 NetworkManager 的命令行工具，主要用于管理网络环境（特别是终端连 Wi-Fi）。它比底层的 `ip` 命令更顶层，是以“连接（Connection）”和“设备（Device）”为核心逻辑的。

```bash
nmcli general status                  # 查看 NetworkManager 运行状态

nmcli device status                   # 查看所有网络接口设备状态 (简写 nmcli d)
nmcli device wifi list                # 扫描环境中所有可用的 Wi-Fi 热点
nmcli device wifi connect "SSID" password "密码"  # 终端直连 Wi-Fi

nmcli connection show                 # 查看所有网络配置 (简写 nmcli c)
nmcli connection show --active        # 只看当前被激活的网络
nmcli connection up "连接名称"         # 启动/重连特定连接（名称通常可用 Tab 补全）
nmcli connection down "连接名称"       # 断开特定连接
nmcli connection reload               # 重载所有网络配置文件
```

### DNS 解析

#### dig

DNS 查询工具，用于调试 DNS 解析问题。是 `nslookup` 的替代品。

```bash
dig example.com               # 查询 A 记录

dig example.com MX            # 邮件记录 (查询指定类型)
dig example.com NS            # 域名服务器
dig example.com CNAME         # 别名记录
dig example.com TXT           # TXT 记录

dig +short example.com        # 简洁输出

dig @8.8.8.8 example.com      # 指定 DNS 服务器查询

dig -x 8.8.8.8                # 反向 DNS 查询

dig +trace example.com        # 追踪 DNS 解析路径
```

### 连通性与路由

#### ping

测试网络连通性和延迟。

> ⚠️ 注意：`ping` 基于 ICMP 协议，在某些网络下可能会被防火墙拦截。同时，许多服务器为防扫描会主动关闭 ping 响应（丢弃 ICMP 请求）。因此，`ping` 不通不能作为判断服务宕机的唯一标准，建议结合 `curl/httpie`、`nc/nmap/telnet` 等工具综合测算端口连通性。

```bash
ping example.com              # 基本 ping
ping -c 4 example.com         # 限制次数
ping -i 0.5 example.com       # 设置间隔时间（秒）
ping -W 2 example.com         # 设置超时时间
ping -s 1472 example.com      # 发送指定大小的包（测试 MTU）
```

#### traceroute

与 `ping` 都是基于 ICMP 协议，追踪数据包到目标主机经过的路由路径。

```bash
traceroute example.com           # 追踪路由
traceroute -I example.com        # 使用 ICMP（某些网络下更可靠）
traceroute -T -p 80 example.com  # 使用 TCP（穿透防火墙）
traceroute -m 20 example.com     # 设置最大跳数
traceroute -n example.com        # 不解析主机名（更快）
```

#### mtr

网络诊断和路由追踪的终极利器。它将 `ping` 和 `traceroute` 的功能结合在一起，动态实时显示每一跳路由的延迟和丢包率，是排查网络节点的首选工具。

```bash
sudo mtr example.com               # 启动交互式动态路由追踪（需要 root 权限）
sudo mtr -n example.com            # 不解析主机名（只显示 IP，防止 DNS 导致解析慢）
sudo mtr -r -c 10 example.com      # 报告模式（非交互）：发 10 个包后直接生成静态统计表
sudo mtr -T -p 443 example.com     # 使用 TCP 协议测试特定端口（穿透 ICMP 防火墙限制）
sudo mtr -i 0.5 example.com        # 设置发包间隔为 0.5 秒（加快测试频率）
```

> 查看 mtr 结果时，以最后一跳的丢包率为准。

#### ping.pe

[ping.pe](https://ping.pe) 是一款全球多节点在线 Ping 与 MTR 路由追踪测试工具。它不仅能快速检测 IP 或域名在全球各地的连通性、延迟与丢包率，更是**排查 GFW（防火长城）网络封禁状态的探针神器**：

- **IP 封禁（强阻断）**：若国内所有探测节点全部飙红（100% 丢包无法连通），而海外节点全绿，说明该 IP 已被重点“照顾”，只能尝试更换 IP。
- **端口封禁（精准阻断）**：若国内节点的 Ping 测试（ICMP 协议）一切正常，但在指定的业务端口连通性测试中惨遭阻断，说明仅端口被墙，为服务更换其他不敏感的端口即可恢复。

> 容器 `/var/lib/docker/containers/[hash_of_the_container]/` 目录下的 `hostconfig.json` 和 `config.v2.json` 记录了容器的网络配置信息，修改此处即可修改容器的端口映射。

#### 终端 IP 归属地查询

纯命令行调用的 IP 地址查询工具，排版极度舒适，常用于**验证当前的终端全局代理（`setproxy`）是否成功生效**。

```bash
# cip.cc (国内最推荐，全中文排版显示 国家/省市/运营商)
curl cip.cc                  # 查本机当前公网 IP 归属
curl cip.cc/111.63.65.247    # 查随便某个特定 IP 的归属

# ipinfo.io (国际通用，原生输出高亮 JSON，包含经纬度/云厂商 ASN，适合脚本处理)
curl ipinfo.io               # 查本机当前公网 IP 归属
curl ipinfo.io/8.8.8.8       # 查特定 IP 归属

# myip.ipip.net (国内公认最准 IP 库 IPIP.net 的纯净单行输出接口)
curl myip.ipip.net
```

### 端口探测与扫描

#### telnet

最经典、最基础的测试目标主机特定 **TCP 端口**是否开放的工具。推荐用 nc 或 nmap 替代。

```bash
telnet example.com 80         # 测试目标 80 端口连通性

# 若连通，会显示 Connected to example.com.
# 若不通，会提示 Connection refused 或一直处于 Trying... (丢包)
#
# 退出当前连接：按下 Ctrl + ] 进入 telnet 提示符，然后输入 quit 并回车
```

#### nmap

强大的网络探测和安全扫描工具，常用于发现网络上的主机和服务，获取主机开放端口、操作系统指纹等。

> 💡 能力覆盖：`nmap` 实际上可以作为 `ping`（仅用于探活时：`-sn`）、`telnet` / `nc`（仅用于测试端口开闭时：`-p`）、`arp`（查询局域网被占用的 IP）的强大平替或进阶补充方案。

```bash
nmap example.com              # 扫描常规的前 1000 个端口
nmap -p 1-65535 192.168.1.1   # 扫描全量端口 (1~65535)
nmap -p 80,443 example.com    # 批量并发扫描离散端口，能区分 open/closed/filtered 状态。相反 telnet 只能单点测试
nmap -sn 192.168.1.0/24       # 只探活（并发 Ping 扫描，扫网段），不扫端口，列出所有存活的 IP 设备
nmap -O 192.168.1.1           # 探测目标操作系统类型（需 root）
nmap -sV example.com          # 探测开放端口的服务及应用版本信息
```

> ⚠️ 防火墙陷阱（False Positive）：很多云服务器的 WAF / 高防网关会对常见端口进行 **“TCP 代理代答”**，导致外网探测（如 nmap、telnet）显示端口 open，但本地 ss -lntp 却查无此进程。 此时可切到服务器终端扫本地环回地址（nmap -p 端口号 127.0.0.1），本地扫出来没开，那就是上游网络设备在代答，系统本身非常安全。

#### nc

`nc`（netcat）是网络调试的瑞士军刀，可用于端口扫描、传输数据、建立连接等。

```bash
nc -zv example.com 80                                                      # 端口扫描
nc -zv example.com 20-100                                                  # 扫描端口范围

nc -l 8080                                                                 # 监听端口（简易服务器）

echo "hello" | nc example.com 80                                           # 发送数据到远程主机

nc -l 9999 > received_file                                                 # 接收端： (传输文件)
nc target_host 9999 < file_to_send                                         # 发送端：

echo -e "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n" | nc example.com 80  # 简易 HTTP 请求
```

### HTTP 通信与传输

#### curl

用于发送 HTTP 请求和传输数据的命令行工具。

```bash
curl https://example.com                                    # GET 请求

curl -I https://example.com                                 # 显示响应头

curl -X POST -H "Content-Type: application/json" \          # POST 请求（发送 JSON）
     -d '{"key":"value"}' https://api.example.com

curl -O https://example.com/file.tar.gz                     # 下载文件
curl -o output.tar.gz https://example.com/file.tar.gz

curl -L https://example.com                                 # 跟随重定向

curl -u user:password https://api.example.com               # 携带认证
curl -H "Authorization: Bearer TOKEN" https://api.example.com

curl --connect-timeout 5 --max-time 15 https://example.com  # 设置超时

curl -F "file=@photo.jpg" https://example.com/upload        # 上传文件

curl -s https://api.example.com | jq '.'                    # 静默模式（只输出响应体）
```

> 💡 现代替代：[httpie](https://github.com/httpie/cli) — 更人性化的语法，自动 JSON 高亮。

```bash
http GET https://api.example.com            # GET 请求
http POST https://api.example.com name=foo  # POST JSON
http -d https://example.com/file.tar.gz     # 下载文件
http -a user:pass https://api.example.com   # 认证
```

#### wget

最经典、且极度坚韧的文件下载利器。与 `curl` 偏向于测试 API 和发送数据不同，`wget` 专精于**大文件下载、断点续传以及整站扒取**。

```bash
wget https://example.com/file.zip             # 基本用法：下载文件到当前目录
wget -O new_name.zip https://example.com/...  # 下载并指定保存的新文件名

wget -c https://example.com/big_file.iso      # 断点续传（如果网络断了，加上 -c 再次运行会接着下载）
wget -b https://example.com/big_file.iso      # 把下载任务全权扔到后台（关闭终端也不受影响）

# 爬虫黑魔法（整站镜像）：
wget -r -p -np -k https://example.com         # 递归下载整个目标网站的所有图片、CSS 和网页，并自动转换成本地链接
```

### 远程传输与备份

#### rsync

`rsync` 降维打击了 `scp` 和 `cp -r`，核心优势是**智能比对哈希差异，只通过网络传输发生变动的几十 KB 碎片（增量同步）**，是处理大文件集群拷贝的终极方案。

```bash
# 基本拷贝（完全替代 cp -r）
rsync -av /src/dir /dest/dir                   # 本地全量同步（完美保留原文件的所有权限、所属用户和创建时间）

# 远程传输（完全替代 scp）
rsync -avP /src/dir user@192.168.1.1:/dest     # 把本地推送到云服务器（P 代表显示进度条并支持断点续传）
rsync -avP user@192.168.1.1:/src /dest         # 把云服务器文件拉取到本地

# 实战杀手锏
rsync -avP --delete /src /dest                 # 真正的镜像级克隆：让目标目录和源完全一致（如果源文件被删，目标库里的也会被强制同步删掉）
rsync -avP --exclude 'node_modules' /src /dest # 自动跳过特定庞大的垃圾黑洞目录
rsync -avP --bwlimit=1000 /src /dest           # 最高网速限制在 1MB/s
```

> **⚠️ 路径结尾带不带 `/` 可是天壤之别！**
> 比如你想传一整个相册：`rsync -a /photo` 会在对面建一个叫 `photo` 的外壳文件夹；而如果多敲了一个斜杠：`rsync -a /photo/`，它就像是“解压缩”一样，只会把相册里的照片全倒在对面的目标地里，不建外壳！

### 终端网络代理

纯底层的终端程序（如 `curl`、`git`、`brew`、`nmap`）默认**不会读取 macOS 系统设置的全局代理或 PAC 规则**。它们极其“固执”，只认当前 Shell 进程内的专属环境变量。

为了更优雅地在使用代理软件（如 Clash）时动态接管终端流量，建议在 `~/.zshrc` (或 bashrc) 中添加以下快速开关别名（假设本地代理软件的混淆端口为默认的 7890）：

```bash
# ~/.zshrc 配置

# 开启终端代理开关
alias setproxy="export all_proxy=socks5://127.0.0.1:7890; export http_proxy=http://127.0.0.1:7890; export https_proxy=http://127.0.0.1:7890; echo '🚀 终端全局代理已开启!'"

# 关闭终端代理开关
alias unsetproxy="unset all_proxy http_proxy https_proxy; echo '🔴 终端全局代理已关闭!'"
```

> **⚠️ 注意**：
> 像 `ping`、`traceroute` 和 `mtr` 等底层排障工具走的是 **ICMP 网络层（L3）协议**。而常见的代理软件一般只能接管 TCP/UDP 即应用层/传输层（L4~L7）的流量。
> 因此**不要指望开启终端代理后能用 `ping` 成功连通外网**，常规代理对 `ping` 是绝对免疫且无效的，测试代理连通性请务必使用 `curl -I google.com` 等 HTTP 替代方案！

## 文件查看

### cat

查看文件内容。

```bash
cat file.txt                          # 查看文件
cat -n file.txt                       # 显示行号
cat file1.txt file2.txt > merged.txt  # 合并多个文件
```

> 💡 现代替代：[bat](https://github.com/sharkdp/bat) — 语法高亮、行号、Git diff 标注、自动分页。

```bash
bat file.py                   # 语法高亮查看
bat -l json file.txt          # 指定语言高亮
bat --diff file.py            # 只显示 Git 变更行
bat -A file.txt               # 显示不可见字符
bat --style=plain file.txt    # 纯文本模式（无行号无边框）
```

### ls

列出目录内容。

```bash
ls -a   # 列出所有文件（含隐藏文件）
ls -l   # 详细信息
ls -lt  # 按时间排序
ls -lS  # 按大小排序
ls -lh  # 格式化大小
```

> 💡 现代替代：[eza](https://github.com/eza-community/eza) — 彩色图标、Git 状态集成、树形视图。

```bash
eza -la                       # 列出所有文件（含隐藏）
eza -l --git                  # 带 Git 状态
eza --tree --level=2          # 树形目录（深度 2）
eza --icons                   # 文件图标
eza -l --sort=modified        # 按修改时间排序
```

> 可用 Rust 实现的 eza 替代 Ruby 实现的 [colorls](https://github.com/athityakumar/colorls)

### yazi

[yazi](https://github.com/sxyazi/yazi) 是当前终端界最火的**全异步、极限性能的 TUI 文件管理器**（同样由 Rust 编写）。它完美继承并彻底超越了老旧的 `ranger`，是现代终端文件管理的最终答案。

> 💡 **神级特性**：原生支持终端内超清渲染图片和视频预览；与 `fzf`、`zoxide`、`rg` 深度融为一体；拥有极其强大的并发大文件拷贝/删除 I/O 能力。

```bash
yazi                          # 启动文件大盘管理器

# 核心实战快捷键：
# h / j / k / l               # 经典 Vim 方向键导航（左下上右）
# <Space>                     # 挑中/取消选中当前文件（支持多选）
# d / y / p                   # 剪切（删）/ 复制 / 粘贴
# .                           # 开启或关闭显示隐藏文件
# Z                           # 无缝唤起 zoxide 全局极速跳转目录
# /                           # 唤起 ripgrep 进行全局代码内容搜索
# f                           # 全键盘极速定位文件（类似于 Vim 的 f 键）
```

## 文本处理

### 匹配与搜索

#### grep

在文件中搜索匹配的文本模式。

```bash
grep "pattern" file.txt                # 基本搜索
grep -r "pattern" /path/to/dir         # 递归搜索目录
grep -i "pattern" file.txt             # 忽略大小写
grep -n "pattern" file.txt             # 显示行号
grep -C 3 "pattern" file.txt           # 显示上下文（前后各 3 行）
grep -v "pattern" file.txt             # 反向匹配（排除）
grep -l "pattern" *.txt                # 只显示文件名
grep -E "error|warning|fatal" log.txt  # 使用正则表达式
grep -c "pattern" file.txt             # 统计匹配次数
```

> 💡 现代替代：[ripgrep (rg)](https://github.com/BurntSushi/ripgrep) — 比 grep 快数倍，默认递归、自动忽略 `.gitignore`。

```bash
rg "pattern"                  # 自动递归搜索当前目录
rg -i "pattern"               # 忽略大小写
rg -t py "import"             # 只搜索 Python 文件
rg -C 3 "error" logs/         # 显示上下文
rg --json "pattern"           # JSON 格式输出
rg -l "TODO"                  # 只显示文件名
rg -g '!test*' "pattern"      # 排除匹配的文件
```

### 提取与切分

#### cut

按列（字段）提取文本。

```bash
cut -d: -f1 /etc/passwd                # 提取用户名 (按分隔符提取指定字段)
cut -d, -f1,3 data.csv                 # 提取第 1 和第 3 列

cut -c1-10 file.txt                    # 提取前 10 个字符 (按字符位置提取)

echo "2024-01-15" | cut -d- -f1        # 提取年份 (与其他命令组合)
ps aux | tr -s ' ' | cut -d' ' -f1,11  # 提取进程用户和命令
```

#### awk

强大的文本处理语言，擅长按列处理结构化文本。

```bash
awk '{print $1}' file.txt                      # 第 1 列 (打印指定列)
awk '{print $1, $3}' file.txt                  # 第 1 和第 3 列
awk '{print $NF}' file.txt                     # 最后一列

awk -F: '{print $1, $3}' /etc/passwd           # 指定分隔符
awk -F, '{print $2}' data.csv

awk '$3 > 100 {print $0}' file.txt             # 第 3 列大于 100 (条件过滤)
awk '/error/ {print $0}' log.txt               # 包含 error 的行
awk 'NR >= 10 && NR <= 20' file.txt            # 第 10-20 行

awk '{sum += $1} END {print sum}' numbers.txt  # 计算求和

awk 'END {print NR}' file.txt                  # 统计行数

awk '{printf "%-20s %s\n", $1, $2}' file.txt   # 格式化输出

awk '!seen[$0]++' file.txt                     # 去除重复（不需要 sort）
```

### 排序与去重

#### sort

对文本行进行排序。

```bash
sort file.txt                 # 默认排序（字典序）
sort -n numbers.txt           # 数值排序
sort -k2 file.txt             # 按第 2 列排序（以空格分隔）
sort -t: -k3 -n /etc/passwd   # 按 UID 排序 (按指定分隔符的字段排序)
sort -r file.txt              # 逆序排序
sort -u file.txt              # 去重排序
du -sh * | sort -h            # 按文件大小排序（支持 K/M/G 后缀）
```

#### uniq

去除或统计连续的重复行（通常与 `sort` 搭配使用）。

```bash
sort file.txt | uniq                # 去除连续重复行
sort file.txt | uniq -d             # 只显示重复行
sort file.txt | uniq -u             # 只显示不重复的行
sort file.txt | uniq -c             # 统计每行出现次数
sort file.txt | uniq -c | sort -rn  # 统计并按出现次数排序
sort file.txt | uniq -i             # 忽略大小写
```

### 统计与对比

#### wc

统计文件的行数、单词数和字节数。

```bash
wc -l file.txt                     # 统计行数
wc -w file.txt                     # 统计单词数
wc -m file.txt                     # 统计字符数
wc -c file.txt                     # 统计字节数
wc file.txt                        # 同时显示全部统计信息
ls | wc -l                         # 统计目录下文件数量
find . -name "*.py" | xargs wc -l  # 统计代码行数
```

#### diff

比较两个文件的差异。

```bash
diff file1.txt file2.txt      # 基本对比
diff -u file1.txt file2.txt   # 统一格式（更易读）
diff -y file1.txt file2.txt   # 并排对比
diff -r dir1/ dir2/           # 对比目录
diff -w file1.txt file2.txt   # 忽略空白差异
```

> 💡 现代替代：[delta](https://github.com/dandavison/delta) — 语法高亮的 diff 工具，可作为 Git pager。

```bash
delta file1.txt file2.txt     # 对比两个文件（语法高亮）

# 作为 git diff 的 pager（在 ~/.gitconfig 中配置）
# [core]
#     pager = delta
# [delta]
#     navigate = true
#     side-by-side = true
```

### 流编辑与替换

#### sed

流编辑器，用于对文本进行替换、删除、插入等操作。

```bash
sed 's/old/new/' file.txt                       # 替换第一个匹配

sed 's/old/new/g' file.txt                      # 替换所有匹配（全局）

sed -i 's/old/new/g' file.txt                   # Linux (原地修改文件（macOS 需要 -i ''）)
sed -i '' 's/old/new/g' file.txt                # macOS

sed '/pattern/d' file.txt                       # 删除匹配行

sed '3d' file.txt                               # 删除第 3 行 (删除指定行)
sed '2,5d' file.txt                             # 删除第 2-5 行

sed '3a\new line content' file.txt              # 在指定行后插入

sed -n '/pattern/p' file.txt                    # 只打印匹配行（类似 grep）

sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt  # 多条命令
```

### 格式化与参数转换

#### jq

命令行 JSON 处理器，用于解析、过滤和转换 JSON 数据。

```bash
echo '{"name":"foo","age":30}' | jq '.'                                              # 格式化 JSON
echo '{"name":"foo","age":30}' | jq '.name'                                          # 提取字段
echo '[1,2,3]' | jq '.[]'                                                            # 遍历数组
echo '[{"name":"a"},{"name":"b"}]' | jq '.[].name'                                   # 提取数组中对象的字段
echo '[{"name":"a","age":20},{"name":"b","age":30}]' | jq '.[] | select(.age > 25)'  # 条件过滤
cat data.json | jq '{username: .name, years: .age}'                                  # 构造新对象
cat data.json | jq '. | length'                                                      # 获取数组长度
curl -s https://api.github.com/users/octocat | jq '.login, .public_repos'            # 与 curl 组合使用
```

#### xargs

将标准输入转换为命令行参数，常配合管道使用。

```bash
echo "file1 file2 file3" | xargs rm                                # 基本用法
cat urls.txt | xargs -I {} curl -O {}                              # 逐行处理（每次传一个参数）
find . -name "*.jpg" | xargs -P 4 -I {} convert {} -resize 50% {}  # 并行执行（同时 4 个进程）
find . -name "*.txt" -print0 | xargs -0 grep "pattern"             # 处理含空格的文件名
echo "file1 file2" | xargs -p rm                                   # 确认执行（交互式）
cat list.txt | xargs -n 2 echo                                     # 限制每次传递的参数数量
```

## 进程与系统

### 系统信息

```bash
lscpu                  # 查看 CPU 架构、核心数等详细物理硬件信息
cat /etc/os-release    # 查看当前 Linux 发行版（如 Ubuntu/CentOS）名称及版本号
uname -a               # 查看内核版本、系统主机名和处理器架构信息
```

### systemctl

`systemctl` 是 `systemd` 的核心指令，用于管理系统服务（Services）、服务自启及其状态大盘。

```bash
systemctl status nginx           # 查看服务当前状态和最近几条启动报错日志
systemctl start nginx            # 启动服务
systemctl stop nginx             # 停止服务
systemctl restart nginx          # 彻底重启服务
systemctl reload nginx           # 平滑重载配置（不中断当前处理，多用于 Nginx 等服务端程序）

systemctl enable nginx           # 设置开机自启
systemctl disable nginx          # 取消开机自启
systemctl mask nginx             # 彻底禁用服务（除非 unmask 解禁，否则该服务无法被任何方式启动）

systemctl is-active nginx        # 脚本常用：检查服务是否在运行（直接返回 active/inactive 等文本）
systemctl is-enabled nginx       # 脚本常用：检查服务是否开机自启

systemctl list-units --type=service --state=running # 查看系统当前正在运行的所有后台服务
systemctl daemon-reload          # 重载所有 unit 配置文件（当你修改了 `/etc/systemd/system/` 下的文件后必须执行）
```

> **💡 最佳搭档 `journalctl`**：由于 systemd 会统一接管所有服务的控制台输出格式，你可以直接用配套工具极速排查日志：
>
> - `journalctl -u nginx -f` ：像 `tail -f` 一样实时滚动查看指定服务的控制台输出。
> - `journalctl -u nginx --since "1 hour ago" -e` ：查看该服务最近一小时的日志并直接翻到最后。

#### 自定义服务（.service 文件）

将自己的程序注册为系统服务，需要在 `/etc/systemd/system/` 下创建一个 `.service` 文件：

```ini
# /etc/systemd/system/my-app.service

[Unit]
Description=My Application          # 服务描述
After=network.target                # 启动顺序：在网络就绪后再启

[Service]
Type=simple                         # 进程类型（simple/forking/oneshot/notify）
User=www-data                       # 以哪个用户身份运行
WorkingDirectory=/opt/my-app        # 工作目录
ExecStart=/opt/my-app/bin/start     # 启动命令（必须写绝对路径）
Restart=on-failure                  # 异常崩溃后自动重启（手动 stop 不触发）
RestartSec=5s                       # 重启前等待时间
Environment="PORT=8080"             # 注入环境变量

[Install]
WantedBy=multi-user.target          # 绑定到多用户运行级别（开机自启的"锚点"）
```

写完后的标准注册流程：

```bash
sudo systemctl daemon-reload          # 必须先重载，让 systemd 识别新文件
sudo systemctl enable --now my-app    # 同时完成"开机自启"和"立即启动"（组合技）
sudo systemctl status my-app          # 验证服务状态
```

#### 定时任务（.timer 文件）

`.timer` 是 `systemd` 的内置定时器，用来替代传统的 `cron`，可以按固定时间或开机后延迟触发另一个 `.service`。`.timer` 和 `.service` 文件**必须同名配对使用**（如 `backup.timer` 驱动 `backup.service`）：

```ini
# /etc/systemd/system/backup.service （被 timer 触发的实际任务）

[Unit]
Description=Daily Backup Task

[Service]
Type=oneshot                        # 运行一次就退出（区别于 simple 的常驻进程）
ExecStart=/opt/scripts/backup.sh
```

```ini
# /etc/systemd/system/backup.timer （定时触发器）

[Unit]
Description=Run backup every day at 3am

[Timer]
OnCalendar=*-*-* 03:00:00           # 每天凌晨 3 点触发（支持 cron 表达式语法）
Persistent=true                     # 若错过触发时间（如关机期间），开机后立即补跑一次

[Install]
WantedBy=timers.target              # 注意：timer 的锚点是 timers.target，而非 multi-user.target
```

常用的 `OnCalendar` 时间表达式：

```
OnCalendar=hourly           # 每小时
OnCalendar=daily            # 每天 00:00
OnCalendar=weekly           # 每周一 00:00
OnCalendar=Mon 09:00        # 每周一 09:00
OnCalendar=*-*-* 03:00:00   # 每天 03:00
OnCalendar=*-*-1 00:00:00   # 每月 1 号
```

> systemd 默认给所有的 Timer 设置了 `AccuracySec=1min`，这意味着：系统保证任务会在 00:00:00 到 00:01:00 这个区间内的某个随机时刻执行，而不是精确在第 0 秒。如果需要精确到秒，可以设置 `AccuracySec=1s`。

启用定时器（只需 enable timer，不需要 enable service）：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer   # 启用并立即激活定时器
systemctl list-timers                      # 查看所有定时器和下次触发时间
```

#### .service vs .timer 选型指南

| 场景                                                 | 选择                                          |
| ---------------------------------------------------- | --------------------------------------------- |
| 需要**持续运行**的后台进程（Web 服务、数据库、代理） | 只写 `.service`                               |
| 需要**定时执行**一次性任务（备份、清理、同步）       | `.service`（Type=oneshot）+ `.timer`          |
| 原本用 `cron` 的场景                                 | 迁移到 `.timer`，可获得日志追踪和错过补跑能力 |

> **💡 为什么用 `.timer` 而不是 `cron`？**
> `systemd timer` 的所有执行记录都会被 `journalctl` 统一收集，可用 `journalctl -u backup.service` 随时追溯历史运行结果；而传统 `cron` 的输出默认只发邮件或直接丢弃，调试极其痛苦。

### sysctl

`sysctl` 用于在运行时读取或修改 Linux 内核参数（位于 `/proc/sys/` 下），是服务器调优和代理/转发类服务部署的必备工具。

```bash
sysctl -a                             # 查看所有内核参数（输出量极大，建议配合 grep 过滤）
sysctl net.ipv4.ip_forward            # 查看指定参数当前值

sysctl -w net.ipv4.ip_forward=1      # 临时修改（立即生效，但重启后恢复原值）
```

> **永久生效**：将参数写入 `/etc/sysctl.d/` 下的配置文件，然后用 `sysctl --system` 重载：
>
> ```bash
> # 推荐：创建独立文件而非直接修改 /etc/sysctl.conf，便于管理和版本追踪
> echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-custom.conf
> sudo sysctl --system   # 重载所有 /etc/sysctl.d/*.conf 配置文件
> ```

常用内核参数速查

```bash
# ── IP 路由转发（部署 VPN / Hysteria2 端口跳跃 / Docker 必须开启）──
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# ── TCP BBR 拥塞控制算法（显著提升高延迟网络的吞吐量）──
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# ── 连接数与文件描述符上限（高并发服务器调优）──
net.core.somaxconn = 65535           # 监听队列最大长度
net.ipv4.tcp_max_syn_backlog = 8192  # SYN 半连接队列长度
fs.file-max = 1000000                # 系统级最大文件描述符数
```

> 应用 BBR 后可用 `sysctl net.ipv4.tcp_congestion_control` 确认是否已切换成功。

### ps

查看正在运行的进程。

```bash
ps aux                          # 查看所有进程（详细信息）
ps aux --sort=-%cpu | head -20  # 按 CPU 使用率排序
ps aux --sort=-%mem | head -20  # 按内存排序
ps auxf                         # 查看进程树
ps aux | grep nginx             # 查找指定进程
```

> 看资源占用率用 `ps aux`，看父子关系链用 `ps -ef`。

> 💡 现代替代：[procs](https://github.com/dalance/procs) — 彩色输出，支持关键字搜索、树形视图。

```bash
procs                         # 查看所有进程
procs --tree                  # 树形视图
procs nginx                   # 按关键字搜索
procs --sortd cpu             # 按 CPU 降序排列
```

### top

实时查看系统资源占用和进程信息。

```bash
top                           # 启动交互式监控

# 按 CPU 排序（默认）
# 按 M 切换为内存排序
# 按 q 退出
```

> 💡 现代替代：[btop](https://github.com/aristocratos/btop) — 美观的全功能系统监控，支持 CPU/内存/网络/磁盘图表。

```bash
btop                          # 启动交互式系统监控
```

> 个人环境推荐使用 `btop`，生产环境可以使用 `top/htop`， 它们不挑剔环境。

## 磁盘管理

### du

查看文件和目录的磁盘占用。

```bash
du -sh .                      # 查看当前目录的总大小
du -sh */                     # 查看各子目录大小
du -sh * | sort -h            # 按大小排序
du -h --max-depth=1 /path     # 查看指定深度
```

> 💡 现代替代：[dust](https://github.com/bootandy/dust) — 可视化条形图展示磁盘占用。

```bash
dust        # 当前目录的磁盘占用可视化
dust -d 2   # 限制深度
dust -r     # 逆序显示
```

### df

查看磁盘分区的使用情况。

```bash
df -h         # 查看所有挂载点的磁盘使用
df -h /home   # 查看指定挂载点
df -Th        # 查看文件系统类型
```

> 💡 现代替代：[duf](https://github.com/muesli/duf) — 表格化彩色展示。

```bash
duf                # 全部磁盘信息
duf /home          # 指定路径
duf --only local   # 只显示本地磁盘
```

### ncdu

`ncdu`（NCurses Disk Usage）是一个终端交互式磁盘占用分析工具，比 `du` 更直观，可以用键盘浏览目录并快速定位占用空间最大的文件。

```bash
ncdu            # 分析当前目录
ncdu /          # 分析根目录
ncdu -x /       # 只分析当前文件系统（不跨挂载点）
ncdu -r /home   # 只读模式（禁止删除操作）
ncdu -e /home   # 显示扩展信息（文件数量）
```

> 界面内按 `d` 可删除选中文件或目录，`?` 查看帮助，`q` 退出。

## 辅助工具

### man

查看命令手册。

```bash
man ls
man 5 passwd   # 查看第 5 章节（文件格式）
```

> 💡 现代替代：[tldr](https://github.com/tldr-pages/tldr) — 简洁实用的命令示例，告别冗长的 man 手册。

```bash
tldr tar    # 查看 tar 的常用示例
tldr curl   # 查看 curl 的常用示例
```

### cd

切换工作目录。

```bash
cd /path/to/dir
cd ~              # 回到家目录
cd -              # 回到上一个目录
```

> 💡 现代替代：[zoxide](https://github.com/ajeetdsouza/zoxide) — 智能目录跳转，自动学习你最常访问的目录。

```bash
z foo      # 跳转到最常用的含 "foo" 的目录
z foo bar  # 匹配同时含 "foo" 和 "bar" 的目录
zi         # 交互式选择（需要 fzf）
```

### zellij

[Zellij](https://github.com/zellij-org/zellij) 是下一代终端多路复用器（Terminal Multiplexer）。完全采用 Rust 编写，旨在取代老旧、配置繁琐的 `tmux`。它开箱即用，带有极其现代化的 UI、原生鼠标支持和悬浮窗功能。

> 💡 **核心优势**：0 学习成本，所有快捷键都常驻显示在底部的状态栏面板上，支持真正的浮动窗口（Floating Panes）以及 WebAssembly 插件生态。

```bash
zellij                          # 启动一个全新的 zellij 会话
zellij attach my-session        # 重新连接到名为 my-session 的后台会话
zellij list-sessions            # 查看所有后台保留运行的会话

# 核心实战快捷键（默认配置下的肌肉记忆）：
# Ctrl + p                      # 呼出面板操作前缀（随后按 n 新建，x 关闭，f 悬浮）
# Ctrl + t                      # 呼出标签页操作前缀（随后按 n 新建标签）
# Ctrl + o                      # 呼出全局会话前缀（随后按 d 把当前会话剥离到后台）
#
# （提示：如果快捷键冲突，可以随时按 Ctrl+g 锁定全部快捷键，纯净输入输入）
```

## 防火墙

<img src="/Linux/imgs/firewall.png" alt="Linux Firewall" style="width: 40%; float: right; margin: 0 0 20px 25px; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.1);">

如图所示，Linux 中提供了多种防火墙工具，最上层是不同 Linux 发行版本提供的防火墙管理工具，可以简单设置防火墙规则；中间是内核的接口层，具有很强的灵活性和精细的控制能力，`iptables` 已经被 `nftables` 取代；最下层是 `netfilter`，是 Linux 内核中的防火墙。

> 注意：不能启动任何两种以上的防火墙，否则防火墙间的冲突会令人很困惑，增加排障难度。

### iptables

`iptables` 底层调用netfilter，但由于 已经被 `nftables` 取代，目前仍有需要系统使用 `iptables`，只需要能够读懂即可。

![iptables](/Linux/imgs/iptables.png)

语法规则：

```bash
iptables [-[ADI] INPUT] [-i lo|eth0|ens3] [-s IP/Netmask] [-d IP/Netmask] [-p tcp|udp [--sport ports] [--dport ports]] [-j ACCEPT|REJECT|DROP]
```

- `-t <table_name>`：指定操作的表（如 `filter`、`nat`、`mangle`），缺省状态下默认是 `filter` 表。
- `-A <chain_name>`：Append，表示在指定的链（如 `INPUT`、`POSTROUTING`）的**最后面**追加一条新规则。同理还有 `-I`（Insert，插入到规则的最前面）和 `-D`（Delete，删除该规则）。
- `-p <protocol>`：匹配特定协议类型，例如 `tcp`、`udp`、`icmp` 等。
- `-s` / `-d`：分别匹配源（Source）IP / 目标（Destination）IP 或网段。
- `--sport` / `--dport`：分别匹配源端口 / 目标端口（使用前通常必须配合 `-p` 指定具体协议）。
- `-i` / `-o`：分别匹配数据包流入的网卡接口（in-interface，如 `eth0`）和流出的网卡接口（out-interface）。
- `-j <target>`：Jump，决定当包被该规则匹配上之后的**最终动作**。常见的有 `ACCEPT`（放行）、`DROP`（直接丢弃报文）、`REJECT`（拒绝并返回一个错误包给发送方）、`SNAT` 以及 `DNAT` 等。

常用命令：

```bash
# 查看规则
iptables -L -v -n                 # 查看所有规则（-v 详细，-n 不解析 DNS）
iptables -L -v -n --line-numbers  # 带规则编号（方便按编号删除）
iptables -t nat -L -v -n          # 查看 NAT 表（端口转发/MASQUERADE）

# 放行端口
iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # 放行 SSH
iptables -A INPUT -p tcp --dport 80 -j ACCEPT    # 放行 HTTP
iptables -A INPUT -p tcp --dport 443 -j ACCEPT   # 放行 HTTPS

# 拒绝/封锁
iptables -A INPUT -s 1.2.3.4 -j DROP             # 封禁指定 IP
iptables -A INPUT -p tcp --dport 3306 -j DROP    # 封锁 MySQL 端口

# 默认策略
iptables -P INPUT DROP      # 默认丢弃所有入站（白名单模式）
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT   # 默认允许所有出站

# 删除规则
iptables -D INPUT 3                              # 按编号删除第 3 条规则
iptables -D INPUT -p tcp --dport 80 -j ACCEPT    # 按内容精确删除
iptables -F                                      # 清空所有规则（⚠️ 危险）
```

### ufw

`ufw`（Uncomplicated Firewall）是 Ubuntu/Debian 系统上的简化防火墙管理工具。

```bash
sudo ufw enable                    # 启用/禁用防火墙
sudo ufw disable

sudo ufw status                    # 查看防火墙状态和规则
sudo ufw status verbose
sudo ufw status numbered           # 带编号显示

sudo ufw default deny incoming     # 默认策略
sudo ufw default allow outgoing

sudo ufw allow 22                  # 允许 SSH (允许/拒绝端口)
sudo ufw allow 80/tcp              # 允许 HTTP
sudo ufw allow 443/tcp             # 允许 HTTPS
sudo ufw deny 3306                 # 拒绝 MySQL

sudo ufw allow 6000:6007/tcp       # 允许端口范围

sudo ufw allow from 192.168.1.100  # 允许指定 IP
sudo ufw allow from 192.168.1.0/24 to any port 22

sudo ufw delete allow 80           # 删除规则
sudo ufw delete 3                  # 按编号删除

sudo ufw reset                     # 重置所有规则
```

### nft

`nft` (nftables) 是 `iptables` 的现代替代品，提供了一个更简单、更高效的结构化规则框架，用来统一管理网络过滤、地址转换（NAT）等功能。

#### 常用命令

```bash
nft list ruleset                       # 查看全系统所有规则（等同于 iptables-save）
nft list tables                        # 查看所有表
nft list table inet filter             # 查看指定表下的规则

nft flush ruleset                      # 清空所有规则（极其危险，相当于关闭所有防火墙过滤）
nft flush table inet filter            # 清空特定表的规则

# 快速临时添加规则（声明一个表 -> 创建一条链 -> 添加规则）
nft add table inet my_filter
nft add chain inet my_filter input { type filter hook input priority 0 \; policy drop \; }
nft add rule inet my_filter input tcp dport 22 accept
```

> **💡 最佳实践**：不要在命令行中一行行敲 `nft` 动态添加规则！`nftables` 最优雅的用法是将其看作类似 Nginx 的配置文件，在一个文件（如 `/etc/nftables.conf`）中写好结构化的规则，然后使用 `nft -f /etc/nftables.conf` 原子级重新加载生效。

#### nft 与 ufw / iptables 的区别

- **ufw (Uncomplicated Firewall)**：本质上是一个**前端包装器**。它的存在是为了让小白用极其简单的命令（如 `ufw allow 22`）来配置防火墙。在较老的系统上它底层调用的是 `iptables`，在较新的 Ubuntu 系统上它底层实际调用的正是 `nftables`。
- **iptables**：古典时代的防火墙霸主。缺点是规则呈现链表式线性匹配（规则越多性能越差），而且 IPv4（`iptables`）和 IPv6（`ip6tables`）必须写两套配置，目前已被 Linux 内核官方标记为过时（废弃）。
- **nftables (nft)**：Linux 官方的新一代网络过滤框架底座。支持在一个 `inet` 表中同时处理 IPv4 和 IPv6 ；支持集合（Sets）和字典映射合并规则；底层原理修改为了类似于伪状态机的字节码，执行和匹配效率极高；语法更加现代化。

#### 注意事项

1. **Docker 穿透防火墙的经典大坑**：由于 Docker 的端口映射逻辑是直接往系统底层的 `NAT 表`（PREROUTING 链）里注入路由转发规则，其优先级处于系统网络栈的最前端。这意味着：**哪怕你用 `ufw deny 80` 封锁了 80 端口，只要你用 `docker run -p 80:80` 启动了容器，外网依然可以畅通无阻地访问该容器！**
   > 解决方案：发布 Docker 服务时，端口映射写死本机回环地址 `127.0.0.1:80:80`（仅允许本地访问），然后通过本机的 Nginx 等反向代理转发并暴露到公网，将访问控制权交还给安全的代理层或原生的主机防火墙。
2. **规则持久化**：直接在终端敲 `nft add` 临时加入的规则保存在内存中，重启系统就会丢失。必须将最终生效的规则通过 `nft list ruleset > /etc/nftables.conf` 导出保存，并确保 `systemctl enable nftables` 服务开机时自动从该文件重载。
3. **不要混用管理工具**：如果你决定用 `ufw`，就彻底忘掉 `iptables` 和 `nft` 命令；如果你觉得 `ufw` 太简陋决定手搓 `nftables.conf`，就必须先把 `ufw` 彻底 `disable`。同时混用两大前端管理工具会导致底层链表交火，轻则网络不通，重则把你自己的 SSH 远程连接锁死在服务器门外。

根据所处的发行版，关闭原有的防火墙管理服务：

```bash
# 1. 停止 ufw (主要针对 Ubuntu/Debian)
# 如果系统没有安装 ufw，报错 "Unit ufw.service not found" 请直接忽略
sudo systemctl stop ufw
sudo systemctl disable ufw

# 2. 停止 iptables 或 firewalld (主要针对 CentOS/RHEL/Fedora)
# 同理，如果 unit 不存在也直接忽略，我们的目的只是确保没有“旧神”在后台干扰
sudo systemctl stop iptables firewalld
sudo systemctl disable iptables firewalld

# 3. 启动 nftables 服务
sudo systemctl start nftables
sudo systemctl enable nftables
```
