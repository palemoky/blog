---
title: 分布式系统中的唯一 ID 设计：从自增主键到 Snowflake 与 UUIDv7
date: 2026-11-06
draft: false
tags: [Distributed Systems, Architecture, Database]
---

单机时代，生成 ID 是一件不需要思考的事：数据库 `AUTO_INCREMENT` 一开，万事大吉。可一旦分库分表、多机房部署，问题就来了——**多台机器各自生成的 ID，怎么保证全局不重复？**

这篇文章梳理一下常见的分布式 ID 方案，以及它们各自的取舍。

---

## 一个好的 ID 需要什么？

不同业务对 ID 的要求差异很大：

| 需求 | 说明 |
| --- | --- |
| **全局唯一** | 底线，不能妥协 |
| **趋势递增** | 对 B+ 树索引友好，写入性能好 |
| **严格单调递增** | 少数场景需要，如消息序号、版本号 |
| **高可用、低延迟** | ID 生成在核心链路上，挂了整个业务都写不进去 |
| **长度适中** | 64 位整数最省空间；128 位次之；字符串最差 |
| **不可预测** | 对外暴露时，不应泄露业务量，也不应能被遍历 |
| **无中心依赖** | 能否在客户端、离线环境本地生成 |

这几个要求之间是互相冲突的。比如“严格递增”和“无中心依赖”基本不可兼得，“趋势递增”和“不可预测”也天然矛盾。**没有完美的方案，只有适合场景的方案。**

---

## 方案一：数据库自增的变体

### 多实例步长

最朴素的思路：N 个数据库实例，每个设置不同的起始值和相同的步长。

```sql
-- 实例 1：1, 4, 7, 10 ...
SET @@auto_increment_offset = 1;
SET @@auto_increment_increment = 3;

-- 实例 2：2, 5, 8, 11 ...
SET @@auto_increment_offset = 2;
SET @@auto_increment_increment = 3;
```

简单，但**扩容极其痛苦**：步长一旦定下就很难改，加一台机器需要重新规划所有实例的 offset。

### 号段模式（Segment）

美团 Leaf 的号段模式是对数据库方案的一次重要改进：**不是每次取一个 ID，而是一次取一段。**

```sql
CREATE TABLE id_alloc (
    biz_tag  VARCHAR(64) PRIMARY KEY,  -- 业务标识，如 order、user
    max_id   BIGINT NOT NULL,          -- 当前已分配的最大 ID
    step     INT    NOT NULL           -- 每次分配的号段长度
);

-- 在同一事务中执行，取到的号段为 (max_id - step, max_id]
UPDATE id_alloc SET max_id = max_id + step WHERE biz_tag = 'order';
SELECT max_id, step FROM id_alloc WHERE biz_tag = 'order';
```

ID 服务把号段缓存在内存里逐个发放，用完再去数据库取下一段。step 设为 1000，数据库压力就降到原来的千分之一。

再进一步，可以用**双 Buffer**：当前号段消耗到一定比例（比如 10%）时，异步预取下一段。这样即使数据库短暂不可用，内存里的号段还能撑一阵子。

**优点**：ID 是纯数字、趋势递增、实现简单，数据库挂了也有缓冲期。
**缺点**：依赖数据库；ID 连续，对外暴露会泄露业务量；服务重启会浪费掉内存里没发完的号段（ID 不连续，但通常无所谓）。

---

## 方案二：UUID

UUID 是 128 位的标识符，最大的特点是**完全不需要协调**，任何机器、任何语言、离线环境都能本地生成。

### UUIDv4：纯随机

v4 有 122 位随机位。碰撞概率有多低？根据生日悖论，要生成约 2.7 × 10¹⁸ 个 v4 UUID，才有 50% 的概率出现一次碰撞。实际工程中可以视为不会重复（前提是用的是密码学安全的随机数源）。

但 v4 有个致命问题：**作为数据库主键时性能很差**。

以 MySQL InnoDB 为例，主键是聚簇索引，数据按主键顺序存放在 B+ 树的叶子节点里。自增 ID 每次都追加在最右侧，页写满了就开新页；而随机 UUID 每次都插到树的随机位置，导致：

- 频繁的**页分裂**，页填充率低，空间浪费；
- 写入位置分散，缓冲池命中率下降，大量随机 I/O；
- 二级索引都要存一份主键，16 字节的主键让所有索引都跟着变大。

### UUIDv7：时间有序的 UUID

2024 年 5 月发布的 [RFC 9562](https://www.rfc-editor.org/rfc/rfc9562) 正式定义了 UUIDv7，可以说是专门为解决上面这个问题而生：

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     unix_ts_ms (48 bits)                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      unix_ts_ms (cont.)       |  ver  |   rand_a (12 bits)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|var|                    rand_b (62 bits)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        rand_b (cont.)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

高 48 位是毫秒级 Unix 时间戳，剩下的是版本号、变体和随机位。于是：

- **按时间趋势递增**，对 B+ 树友好，写入性能接近自增 ID；
- **仍然无需协调**，本地即可生成；
- 74 位随机数保证同一毫秒内也几乎不会碰撞；
- 是标准 UUID 格式，现有的 `uuid` 类型字段可以直接用。

生态也已经跟上了，比如 PostgreSQL 18 内置了 `uuidv7()` 函数。**如果你今天在新项目里想用 UUID 作主键，直接用 v7，别再用 v4 了。**

需要注意的是，v7 暴露了生成时间，对时间敏感的场景（比如不希望别人知道用户何时注册）要权衡一下。

---

## 方案三：Snowflake

Twitter 在 2010 年开源的 Snowflake，是分布式 ID 领域最经典的设计。它把一个 64 位整数切成几段：

```text
 1 bit        41 bits                10 bits         12 bits
+---+--------------------------+----------------+--------------+
| 0 |     毫秒时间戳（相对纪元）   |    机器 ID     |   序列号      |
+---+--------------------------+----------------+--------------+
```

- **符号位**：恒为 0，保证 ID 是正数；
- **41 位时间戳**：相对于自定义纪元（epoch）的毫秒数，2⁴¹ 毫秒约等于 **69 年**；
- **10 位机器 ID**：最多 1024 个节点，常拆成 5 位机房 + 5 位机器；
- **12 位序列号**：同一毫秒内的自增序号，单节点每毫秒最多 4096 个，即每秒约 409 万个。

它的精妙之处在于：**把“全局唯一”这个分布式问题，拆解成了“机器 ID 唯一”和“单机内唯一”两个子问题。** 前者只需在部署时分配一次，后者在单机内用一把锁就能解决。生成 ID 完全在本地完成，不需要任何网络调用。

### 一个最小实现（Go）

```go
package snowflake

import (
	"errors"
	"sync"
	"time"
)

const (
	epoch      int64 = 1704067200000 // 2024-01-01 00:00:00 UTC
	workerBits       = 10
	seqBits          = 12

	maxWorker = (1 << workerBits) - 1 // 1023
	maxSeq    = (1 << seqBits) - 1    // 4095

	workerShift = seqBits
	timeShift   = seqBits + workerBits
)

var ErrClockBackwards = errors.New("snowflake: clock moved backwards")

type Node struct {
	mu       sync.Mutex
	workerID int64
	lastMs   int64
	seq      int64
}

func New(workerID int64) (*Node, error) {
	if workerID < 0 || workerID > maxWorker {
		return nil, errors.New("snowflake: worker id out of range")
	}
	return &Node{workerID: workerID}, nil
}

func (n *Node) Next() (int64, error) {
	n.mu.Lock()
	defer n.mu.Unlock()

	now := time.Now().UnixMilli()

	// 时钟回拨：小幅回拨等一等，大幅回拨直接报错
	if now < n.lastMs {
		if n.lastMs-now <= 5 {
			time.Sleep(time.Duration(n.lastMs-now) * time.Millisecond)
			now = time.Now().UnixMilli()
		}
		if now < n.lastMs {
			return 0, ErrClockBackwards
		}
	}

	if now == n.lastMs {
		n.seq = (n.seq + 1) & maxSeq
		if n.seq == 0 { // 本毫秒的序列号用完了，自旋等到下一毫秒
			for now <= n.lastMs {
				now = time.Now().UnixMilli()
			}
		}
	} else {
		n.seq = 0
	}

	n.lastMs = now
	return (now-epoch)<<timeShift | n.workerID<<workerShift | n.seq, nil
}
```

### Snowflake 的几个坑

**1. 时钟回拨**

Snowflake 强依赖本机时钟。NTP 校时、人为改时间、虚拟机迁移都可能让时钟倒退，一旦倒退就可能生成重复 ID。常见处理方式：

- 小幅回拨（几毫秒）：像上面的代码一样等待追平；
- 大幅回拨：拒绝服务并告警，摘掉该节点；
- 更激进的做法：启动时把上次的时间戳持久化，或者干脆不用实时时钟，改用“逻辑时钟”（如百度 UidGenerator 借用未来时间的做法）。

**2. 机器 ID 怎么分配**

10 位机器 ID 必须全局唯一。在容器化环境里，Pod 随时会漂移、重建，手动配置不现实。常见做法：

- 用 ZooKeeper / etcd 的顺序节点或租约分配（Leaf-snowflake 就是这么做的）；
- 用数据库表登记“机器 → workerID”的映射；
- Kubernetes 里用 StatefulSet 的序号。

千万别用 IP 地址的后几位或者随机数凑，碰撞了很难排查。

**3. 低并发下的尾数不均**

如果业务并发很低，每毫秒只产生一个 ID，那序列号永远是 0，**ID 的低 12 位全是 0**。这时候如果按 `id % 分片数` 分库分表，数据会全部落到同一个分片上。解决办法是每毫秒的序列号从一个随机值开始，或者分片时用哈希而不是直接取模。

**4. JavaScript 精度问题**

JavaScript 的 `Number` 是双精度浮点数，只能精确表示 2⁵³ 以内的整数，而 Snowflake ID 能到 2⁶³。后端返回的 ID 到了前端可能尾数被改掉，导致“明明有这条数据却查不到”。Twitter 当年的解法是 API 里同时返回 `id` 和 `id_str`。**最稳妥的做法：对外接口里，64 位 ID 一律序列化成字符串。**

### 变体们

| 方案 | 时间位 | 机器位 | 序列位 | 特点 |
| --- | --- | --- | --- | --- |
| Twitter Snowflake | 41（毫秒） | 10 | 12 | 原版 |
| 百度 UidGenerator | 28（秒） | 22 | 13 | 支持更多机器重启；RingBuffer 预生成 |
| 美团 Leaf-snowflake | 41（毫秒） | 10 | 12 | ZooKeeper 分配 workerID、检查时钟 |
| MongoDB ObjectId | 32（秒） | 40 位随机值 | 24 | 96 位，客户端生成 |

可以看到，**本质都是在“时间范围、节点数量、单节点吞吐”三者之间分配比特位**。业务部署规模小，就可以把机器位让给序列号；需要用很多年，就把时间精度降到秒。

---

## 安全视角：ID 不只是 ID

很多人设计 ID 时只考虑唯一性和性能，忽略了 ID 一旦对外暴露就会**泄露信息**：

- **业务量**：订单号是连续自增的，竞争对手早晚各下一单，两个订单号一减就知道你一天的订单量。二战时盟军就是靠缴获德军坦克的序列号估算出了坦克产量，这就是著名的“德国坦克问题”；
- **可遍历**：`/api/users/10086` 能访问，攻击者会顺手试试 10087、10088……如果接口的权限校验有疏漏（IDOR，越权访问），就是一次数据泄露；
- **生成时间**：Snowflake 和 UUIDv7 都可以反解出生成时间。

常见做法是**内外两套 ID**：内部用 Snowflake / 自增做主键，保证性能；对外暴露的用 UUIDv4 或对内部 ID 做可逆加密（如 Hashids、Sqids 这类编码或格式保留加密）。当然，**ID 不可预测永远不能代替权限校验**，它只是多一道防线。

---

## 怎么选

| 场景 | 推荐 |
| --- | --- |
| 单库、小规模业务 | 数据库自增就够了，别过度设计 |
| 需要 64 位整数、高吞吐、趋势递增 | Snowflake（及其变体） |
| 强依赖数据库也能接受，希望 ID 简单连续 | 号段模式 |
| 客户端/离线生成、多系统数据合并 | UUIDv7 |
| 对外暴露、不能泄露任何信息 | UUIDv4，或内部 ID 加密后对外 |
| 严格单调递增（如消息序号） | 单点发号器 + 主备，或按分区内单调 |

---

## 小结

分布式 ID 的各种方案，归根结底都在回答同一个问题：**如何在不（或尽量少）协调的情况下，让多个节点生成的值不冲突。**

- 号段模式：**用中心化换简单**，但把协调频率降到了最低；
- UUID：**用空间换去中心化**，靠概率保证唯一；
- Snowflake：**用结构换去中心化**，把唯一性拆到时间、机器、序列三个维度上，代价是依赖时钟和机器 ID 分配。

理解了这几种取舍，面对新的方案（以及未来的新变体）时，也就能快速判断它适不适合自己的场景了。

---

**参考资料**

- [RFC 9562: Universally Unique IDentifiers (UUIDs)](https://www.rfc-editor.org/rfc/rfc9562)
- [Announcing Snowflake - Twitter Engineering Blog (2010)](https://blog.x.com/engineering/en_us/a/2010/announcing-snowflake)
- [Leaf——美团点评分布式 ID 生成系统](https://tech.meituan.com/2017/04/21/mt-leaf.html)
- [baidu/uid-generator](https://github.com/baidu/uid-generator)
- [ULID Spec](https://github.com/ulid/spec)
