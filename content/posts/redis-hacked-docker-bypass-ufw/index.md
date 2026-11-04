---
title: Docker 端口绕过 ufw：一次 Redis 被入侵的复盘
date: 2026-11-04
draft: false
tags: [安全, Docker, Redis]
---

## 起因：一个从没设过的密码

我的斗地主服务 [fight-the-landlord](https://github.com/palemoky/fight-the-landlord) 突然起不来了，poker 容器反复报错退出，日志说 Redis 需要认证。可我从来没给 Redis 设过密码，配置也半年没动过。

排查下来的结论是：**Redis 被公网扫描器入侵了，密码是攻击者设的。** 而 Redis 之所以暴露在公网，是因为 Docker 发布的端口会绕过 ufw——我以为防火墙挡住了，其实它从来没挡过。

## 排查：日志里的三条线索

Redis 的日志把整件事记得清清楚楚。先确认现象：容器启动参数只有 `redis-server --appendonly yes`，没有密码，但 `redis-cli PING` 返回 `NOAUTH Authentication required.`。密码是运行时被人用 `CONFIG SET requirepass` 设上的，这个命令不会写日志。

**线索一：大量跨协议探测。** 日志里有 144 条来自不同公网 IP 的告警：

```
# Possible SECURITY ATTACK detected. It looks like somebody is sending POST or Host:
commands to Redis. ... Connection from 91.230.168.159:53549 aborted.
```

这说明 6379 端口一直在被全网扫描。

**线索二：有人把我的 Redis 变成了别人的从库。** 4 月 1 日就有一次，容器启动才第二天：

```
REPLICAOF 203.57.109.214:60114 enabled (user request from
  'addr=165.154.41.120:40840 laddr=172.19.0.2:6379 ... cmd=slaveof')
Full resync from master: ZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZ:0
MASTER <-> REPLICA sync: receiving 21816 bytes from master to disk
```

10 月 29 日又来一次，这次的关键一行是：

```
# Wrong signature trying to load DB from file
# Failed trying to load the MASTER synchronization DB from disk
```

**线索三：又被切回了主库。** 一秒后：

```
MASTER MODE enabled (user request from 'addr=106.14.68.206:52302
  laddr=172.19.0.2:6379 age=7 ... cmd=slaveof ... tot-cmds=11')
```

注意 `addr` 是公网 IP。如果攻击是经由 poker 发起的，这里应该是 poker 容器的内网地址 `172.19.0.x`。所以攻击者是直接连上 Redis 的，和 poker 代码是否开源无关。

## 攻击原理：rogue master

这是针对未加密码 Redis 的经典攻击，利用的是主从复制功能。自动化脚本的流程大致如下：

1. 扫描全网 6379，找到不需要密码的 Redis。
2. 发送 `SLAVEOF <攻击者IP> <端口>`，让受害者变成自己的从库。
3. 伪造的主库宣称要做全量同步，把一个恶意的 `.so` 文件当作 RDB 发过来，受害者会把它写到磁盘。
4. 执行 `MODULE LOAD` 加载这个文件，就能在 Redis 进程里执行任意系统命令。
5. 发送 `SLAVEOF NO ONE` 切回主库，清理痕迹。

对照日志：`Full resync from master: ZZZZ...:0` 是伪造主库的典型特征，正常主库不会发这种 replication ID。`Wrong signature` 说明收到的不是合法 RDB，这符合传恶意模块的特征，但我没有拿到文件本身。

那条切回主库的连接 `age=7`、`tot-cmds=11`，已经存在 7 秒、执行过 11 条命令。很可能是同一个脚本从头到尾完成了第 2 步和第 5 步，伪造主库和控制连接用的是不同 IP。

**这次攻击应该是卡在了第 4 步。** `Wrong signature` 本身在攻击者预期之内：载荷本来就不是 RDB，攻击者只需要它落盘。真正的护栏是 Redis 7.0 起默认的 `enable-module-command no`，`MODULE LOAD` 直接被拒绝，`dir`、`dbfilename` 等保护配置也不能用 `CONFIG SET` 修改。日志里从没出现过官方以外的 `Module loaded`，`/data` 里也没有可疑文件。但同步前 Redis 已经执行了 `Flushing old data`，原有数据被清空了。

## 根因：Docker 的端口绕过了 ufw

我的 ufw 没有放行 6379，但从外网 `printf 'PING\r\n' | nc 服务器IP 6379` 直接拿到了 `-NOAUTH` 回复。问题不在 ufw 的规则，而在数据包根本没经过 ufw 管的那条链。

![数据包路径：宿主机进程 vs Docker 发布的端口](docker-ufw-netfilter.png)

写下 `ports: "6379:6379"` 时，Docker 会在 `nat` 表的 PREROUTING 阶段加一条 DNAT 规则，把发往宿主机 6379 的包的目标改写成容器 IP `172.19.0.2:6379`。这一步发生在路由判断之前。

目标变成容器后，内核认为这个包要转发出去，它走 **FORWARD** 链，不走 **INPUT** 链。而 ufw 的入站规则主要在 INPUT 链上。Docker 还在 FORWARD 链最前面插入了自己的放行规则（`DOCKER-USER`、`DOCKER-FORWARD` 等）。

所以这不是“Docker 优先级更高”，而是 ufw 压根没机会检查这些包。`ufw status` 里看不到 6379，可它从 3 月 31 日容器启动起就一直对全网开放，第二天就被攻击了。

反过来，host 网络模式的容器和宿主机进程一样走 INPUT 链，是受 ufw 管的。想让防火墙管住 Docker 发布的端口，规则要加在 Docker 留给用户的 `DOCKER-USER` 链里，或者用 [ufw-docker](https://github.com/chaifeng/ufw-docker) 这类工具。

## 影响评估

攻击没有拿到代码执行，损失是一次数据清空和一次服务中断。我从下面几个方向确认了一遍：

| 检查项     | 结果                                                       |
| ---------- | ---------------------------------------------------------- |
| Redis 模块 | `enable-module-command` 为 `no`，只加载过官方自带的 4 个模块 |
| 容器进程   | 只有一个 `redis-server`，以 uid 999 运行，没有挖矿进程     |
| 容器挂载   | 只有数据卷 `/data`，没有挂宿主机目录                       |
| AOF 内容   | 只有房间、战绩、排行榜，没有 cron 或 SSH 公钥载荷          |
| 宿主机     | CPU 正常，`/etc/cron.d` 没有异常，没有可疑外连             |
| 数据       | 10 月 29 日之前的数据被清空                                |
| 服务       | 密码被设上后，poker 无法连接，反复报错退出                 |

还有一个隐蔽问题：Redis 返回 `NOAUTH` 时，`redis-cli ping` 的退出码仍是 0。所以整个过程中容器一直显示 `healthy`，healthcheck 没能发现问题。

## 修复

核心只有一条：**Redis 不映射任何端口。** poker 和 redis 在同一个 Docker 网络里，用服务名 `redis:6379` 就能访问。顺便一提，容器里的 `localhost` 指的是容器自己，不是宿主机。

再加上密码作为第二道防线。密码放在 compose 项目根目录的 `.env` 里，redis 和 poker 都从这里取：

```bash
echo "REDIS_PASSWORD=$(openssl rand -hex 24)" >> .env
chmod 600 .env
```

```yaml
  poker:
    environment:
      - REDIS_PASSWORD=${REDIS_PASSWORD}

  redis:
    image: redis:8-alpine
    # 注意：没有 ports
    environment:
      - REDISCLI_AUTH=${REDIS_PASSWORD}   # 让容器内的 redis-cli 自动带上密码
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD:?REDIS_PASSWORD 未设置}
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep -q PONG"]
```

几个细节：

- **healthcheck 必须检查 `PONG`**，不能只看退出码，原因见上一节。
- **`command` 要以 `redis-server` 开头**。官方镜像的入口脚本只在这种情况下切换到 redis 用户，改成 `sh -c ...` 会变成用 root 运行。
- **用十六进制密码**，避免特殊字符在 shell 或 URL 里出问题。

然后重建容器。攻击者设的密码只存在于内存里，重建后就没了：Redis 没有配置文件，`CONFIG REWRITE` 写不进去。

```bash
docker compose up -d --force-recreate redis
docker port poker-redis                  # 应无输出
docker exec poker-redis redis-cli ping   # 应返回 PONG
docker compose up -d poker
```

最后，我把 [开源仓库](https://github.com/palemoky/fight-the-landlord) 里 `docker-compose.yml` 的 Redis 端口映射也删了，避免别人照着部署时踩同一个坑。

## 举一反三

### 把所有容器的端口都查一遍

既然 ufw 管不到 Docker，compose 里每一个 `"端口:端口"` 都是对公网开放的。我用这两条命令把所有暴露的端口列了出来：

```bash
docker ps --format '{{.Names}}\t{{.Ports}}'          # 正在运行的
docker compose config | grep -E 'published|host_ip'  # 包括没启动的
```

又发现了两个没启动但一启动就会出事的服务：PostgreSQL 的 `5432` 和 frps 管理面板的 `7501`。以及一个 host 网络模式下监听 `*:1984` 的 hysteria 流量统计 API，它本来只需要本机访问。原则很简单：

- 只给内部用的：不写 `ports:`，走 Docker 网络。
- 只给宿主机用的：写成 `"127.0.0.1:端口:端口"`。
- 确实要对外的：确认有认证，并知道 ufw 拦不住它。

### 验证时被 Clash 骗了

修复后我在本机跑 `nc -z 服务器IP 6379`，竟然还是 `succeeded`。但服务器上 `docker port` 和 `ss -tln` 都显示没有任何东西在监听 6379。

问题在本机：Clash 开着 TUN 模式，到服务器的路由走的是 `utun` 虚拟网卡。TUN 会先在本地完成 TCP 握手，再去连真正的目标，所以 `nc -z` 测什么端口都是成功。拿一个肯定没开的端口 6390 一测，也是 `succeeded`。

更可靠的测法是发一条真实的命令，看有没有回应：

```bash
printf 'PING\r\n' | nc -w 5 服务器IP 6379
```

修复前它返回 `-NOAUTH Authentication required.`，修复后没有任何输出。或者干脆关掉代理、换一个网络来测。

## 总结

这次事故是三个问题叠在一起：Redis 映射了端口、没有密码、我以为 ufw 能挡住。任何一个不成立，都不会出事。让它没有变成服务器失陷的，是 Redis 7 以后默认关闭的 `MODULE` 命令。

留给自己的清单：

- `ports: "6379:6379"` 就是对全网开放，ufw 和 firewalld 都管不到。
- 数据库、缓存这类内部服务不映射端口；只给本机用就绑 `127.0.0.1`。
- 内网服务也要设密码，它是网络配置出错时的最后一道防线。
- healthcheck 要检查返回内容，不要只看退出码。
- 想用防火墙管 Docker 端口，规则要加在 `DOCKER-USER` 链。
- 测端口时先确认本机没有开 TUN 代理，并拿一个关闭的端口对照。
- 服务突然出问题而自己“什么都没改”，先怀疑是不是别人改了。
