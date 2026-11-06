---
title: 基于 Merkle Tree 的对账系统：只比较不一样的部分
date: '2026-11-06T13:03:50+08:00'
draft: false
tags: [Distributed Systems, Architecture, Database]
---

做过支付的人都知道，**对账**是资金安全的最后一道防线。前面几篇聊了[订单号]({{< relref "/posts/distributed-order-number/index.md" >}})和[幂等]({{< relref "/posts/distributed-idempotency/index.md" >}})，它们都在尽量保证“不出错”；而对账要解决的是另一个问题：**万一出错了，能不能及时发现？**

这篇文章介绍一种用 Merkle Tree 做对账的思路：当两边的数据量很大、而差异又很少时，它能让我们只比较不一样的那一小部分。

---

## 对账在对什么

对账就是把两份**本应一致**的数据放在一起逐条比较，找出不一致的地方。常见的对账场景有：

- **外部对账**：本地的支付单和支付渠道每天提供的对账单；
- **内部对账**：订单系统、支付系统、账务系统之间，比如“已支付的订单”和“支付成功的支付单”是否一一对应；
- **跨机房、跨库**：主备库之间、数据迁移前后的数据是否一致。

比较的结果，无外乎这几类差错：

| 差错类型 | 含义 | 典型原因 |
| --- | --- | --- |
| 本地有、对端无 | 我们认为成功了，对端没有记录 | 本地状态被错误更新；对端还没入账（跨日） |
| 对端有、本地无 | 钱已经收到，本地却没有成功记录 | 支付回调丢失，俗称“掉单” |
| 金额不一致 | 两边都有，金额不同 | 部分退款未同步、手续费口径不同、代码 bug |
| 状态不一致 | 两边都有，状态不同 | 退款、关单等后续操作只在一边生效 |

其中“本地有、对端无”最危险：我们可能已经发了货，钱却没有到账。“对端有、本地无”也很常见：用户付了钱，订单却一直显示未支付，用户会直接投诉。

---

## 传统做法：全量比较

最直接的做法是每天（T+1）把两边的数据都拉到一起，按主键做一次全外连接：

```sql
SELECT l.pay_no, l.amount, r.amount, l.status, r.status
FROM local_payments l
FULL OUTER JOIN channel_bill r ON l.pay_no = r.pay_no
WHERE l.pay_no IS NULL
   OR r.pay_no IS NULL
   OR l.amount <> r.amount
   OR l.status <> r.status;
```

> MySQL 不支持 `FULL OUTER JOIN`，需要用 `LEFT JOIN` 加 `UNION` 来模拟；数据量更大时一般会放到 Spark、Hive 这类离线计算平台上去跑。

这个方案简单可靠，数据量不大时完全够用。但随着规模增长，问题也逐渐暴露：

- **要把全量数据搬到一起**：两份数据分别在不同的系统、不同的库，甚至不同的机房，每天都要把几千万条记录完整地传一遍；
- **比较成本和数据量成正比**：哪怕两边只有 3 条不一致，也得把几千万条全部比一遍；
- **重跑成本高**：处理完差错要复核一次，或者对端补发了数据，又要从头全量比一遍。

但是仔细想想，对账数据有一个很明显的特点：**绝大部分数据都是一致的，差异极其稀疏。** 如果能快速判断“哪些部分是一致的”并直接跳过，只比较可能不一致的部分，效率就能大大提高。这正是 Merkle Tree 擅长的事情。

---

## Merkle Tree 是什么

Merkle Tree（默克尔树，也叫哈希树）是一棵每个节点都存着哈希值的树：

- **叶子节点**存的是数据块的哈希；
- **内部节点**存的是它所有子节点哈希拼接后再求的哈希；
- 最终汇聚到唯一的**根节点**。

```text
                  root = H(H12 ‖ H34)
                 /                    \
        H12 = H(H1 ‖ H2)        H34 = H(H3 ‖ H4)
          /        \              /        \
     H1=H(D1)   H2=H(D2)     H3=H(D3)   H4=H(D4)
```

它有一个非常有用的性质：**任何一个数据块发生变化，从它到根节点路径上的所有哈希都会跟着变。** 反过来，只要两棵树的根相同，就可以认为两边的数据完全一致（用 SHA-256 这样的哈希函数时，碰撞的概率可以忽略不计）。

于是，比较两份数据就变成了从根开始逐层向下比较两棵树：

1. 根相同：数据完全一致，结束；
2. 根不同：比较它的子节点，只深入哈希不同的那些子树；
3. 一直到叶子层，就精确定位到了哪些数据块不一致。

这个思路在分布式系统里早就被广泛使用：Amazon 的 Dynamo 和 Cassandra 用它在副本之间做反熵（anti-entropy）修复，Git 用类似的哈希结构判断哪些对象需要同步，比特币用它组织区块里的交易，证书透明度日志（[RFC 6962](https://www.rfc-editor.org/rfc/rfc6962)）用它证明证书确实被记录在日志中。

---

## 设计关键：树的形状必须由主键决定

把 Merkle Tree 用到对账上，第一个要解决的问题是：**叶子怎么划分？**

最容易想到的做法是把记录按主键排序，每 N 条作为一个叶子。但这样有个致命问题：只要对端**多一条或少一条**记录，它之后所有记录的位置都会整体错开一位，几乎所有叶子的哈希都会变化，树的比较就退化成了全量比较。

正确的做法是**让每条记录落在哪个叶子上，只由它的主键决定**，与记录的总数、插入顺序都无关。具体来说：

- 预先固定叶子（桶）的数量，比如 $2^{16} = 65536$ 个；
- 对主键做哈希，取前 16 位作为桶编号；
- 每个叶子的哈希由桶内的所有记录计算得出。

这样，多一条、少一条、改一条记录，都**只会影响它所在的那一个桶**，以及从这个桶到根的那条路径。

桶的数量是一个权衡：100 万条记录分到 65536 个桶里，平均每桶约 15 条，定位到桶之后逐条比较的成本很低；整棵树有 $2^{17} - 1$ 个节点，每个哈希 32 字节，总共约 4 MB，存储和计算都很轻松。数据量再大一个数量级，可以把树再加高几层。

### 按对账日分树

实际系统中，通常会**每个对账维度建一棵树**，比如“某个支付渠道 × 某个商户号 × 某一天”。分树的好处是：

- 每天的树相互独立，某一天的数据有变动，不影响其他日期；
- 可以把每天的根哈希存档下来。将来复核时，只要重新计算并比较根哈希，就能快速确认这一天的账有没有被改动过。

需要注意的是，**对账日要以资金实际发生的时间为准**（比如支付成功时间），而不是订单的创建时间，并且两边要使用同一个时区和日切口径，后面会详细说到跨天的问题。

---

## 记录的规范化

Merkle Tree 比较的是哈希，哈希对输入极其敏感：差一个字节，结果就完全不同。所以在计算哈希之前，两边必须把记录转换成**完全相同的规范形式**：

- **只包含双方都必须一致的字段**：主键、金额、状态。像本地的备注、更新时间这类字段不应该参与计算；
- **金额统一用整数“分”表示**：`1.0`、`1.00` 和 `1` 在业务上相同，但字符串完全不同，浮点数还有精度问题；
- **状态映射到统一的枚举**：渠道的 `TRADE_SUCCESS` 和本地的 `PAID`，要先映射成同一个值；
- **编码要无歧义**：直接把字段拼接起来是不行的，`"ab" + "c"` 和 `"a" + "bc"` 拼出来是一样的。可以在每个字段前加上长度；
- **叶子和内部节点要区分**：计算叶子哈希时加前缀 `0x00`，计算内部节点哈希时加前缀 `0x01`，这样叶子的哈希和内部节点的哈希就不会被混淆。RFC 6962 采用的就是这种做法。

---

## 实现（Go）

下面是一个完整的最小实现，包括建树、比较和桶内逐条对比：

```go
package recon

import (
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"sort"
)

const Depth = 16 // 树高 16，共 2^16 = 65536 个桶

type Hash = [32]byte

// Record 是参与对账的一条记录，只包含双方都必须一致的字段
type Record struct {
	OrderNo string // 对账主键，如支付单号
	Amount  int64  // 金额，统一用“分”表示，避免浮点数
	Status  string // 如 SUCCESS、REFUNDED
}

// encode 规范化编码：每个字段前加 4 字节长度，避免 "ab"+"c" 与 "a"+"bc" 编码相同
func (r Record) encode() []byte {
	var buf bytes.Buffer
	amount := binary.BigEndian.AppendUint64(nil, uint64(r.Amount))
	for _, f := range [][]byte{[]byte(r.OrderNo), amount, []byte(r.Status)} {
		buf.Write(binary.BigEndian.AppendUint32(nil, uint32(len(f))))
		buf.Write(f)
	}
	return buf.Bytes()
}

// Bucket 只由主键决定，与记录的数量、插入顺序都无关
func Bucket(orderNo string) int {
	h := sha256.Sum256([]byte(orderNo))
	return int(binary.BigEndian.Uint16(h[:2])) // 取前 16 位，对应 Depth = 16
}

type Tree struct {
	levels  [][]Hash   // levels[0] 是叶子层，levels[Depth] 只有根节点
	buckets [][]Record // 每个桶里的原始记录
}

func Build(records []Record) *Tree {
	t := &Tree{buckets: make([][]Record, 1<<Depth)}
	for _, r := range records {
		b := Bucket(r.OrderNo)
		t.buckets[b] = append(t.buckets[b], r)
	}

	leaves := make([]Hash, 1<<Depth)
	for i := range t.buckets {
		leaves[i] = leafHash(t.buckets[i])
	}
	t.levels = [][]Hash{leaves}
	for lv := leaves; len(lv) > 1; {
		next := make([]Hash, len(lv)/2)
		for i := range next {
			next[i] = nodeHash(lv[2*i], lv[2*i+1])
		}
		t.levels = append(t.levels, next)
		lv = next
	}
	return t
}

func (t *Tree) Root() Hash { return t.levels[Depth][0] }

// Nodes 批量返回某一层若干节点的哈希，对端通常以 RPC 接口的形式提供
func (t *Tree) Nodes(level int, idx []int) []Hash {
	out := make([]Hash, len(idx))
	for i, j := range idx {
		out[i] = t.levels[level][j]
	}
	return out
}

func (t *Tree) Records(bucket int) []Record { return t.buckets[bucket] }

// 叶子和内部节点使用不同的前缀（0x00 / 0x01），避免两者的哈希被混淆
func leafHash(rs []Record) Hash {
	sort.Slice(rs, func(i, j int) bool { return rs[i].OrderNo < rs[j].OrderNo })
	h := sha256.New()
	h.Write([]byte{0x00})
	for _, r := range rs {
		e := r.encode()
		h.Write(binary.BigEndian.AppendUint32(nil, uint32(len(e))))
		h.Write(e)
	}
	var out Hash
	copy(out[:], h.Sum(nil))
	return out
}

func nodeHash(l, r Hash) Hash {
	h := sha256.New()
	h.Write([]byte{0x01})
	h.Write(l[:])
	h.Write(r[:])
	var out Hash
	copy(out[:], h.Sum(nil))
	return out
}

type NodeSource interface {
	Nodes(level int, idx []int) []Hash
}

// Diff 从根开始逐层向下比较，只展开哈希不一致的节点，返回不一致的桶编号。
// 每层只向对端发一次批量请求，总共 Depth+1 次往返。
func Diff(local, remote NodeSource) []int {
	cand := []int{0}
	for level := Depth; ; level-- {
		l, r := local.Nodes(level, cand), remote.Nodes(level, cand)
		var diff []int
		for i, idx := range cand {
			if l[i] != r[i] {
				diff = append(diff, idx)
			}
		}
		if level == 0 || len(diff) == 0 {
			return diff
		}
		cand = make([]int, 0, 2*len(diff))
		for _, idx := range diff {
			cand = append(cand, 2*idx, 2*idx+1)
		}
	}
}

type Kind string

const (
	OnlyLocal  Kind = "本地有、对端无"
	OnlyRemote Kind = "对端有、本地无"
	Different  Kind = "字段不一致"
)

type Mismatch struct {
	Kind          Kind
	Local, Remote *Record
}

// CompareBucket 逐条比较同一个桶里的记录，桶很小，直接用 map 即可
func CompareBucket(local, remote []Record) []Mismatch {
	idx := make(map[string]*Record, len(remote))
	for i := range remote {
		idx[remote[i].OrderNo] = &remote[i]
	}
	var out []Mismatch
	for i := range local {
		l := &local[i]
		r, ok := idx[l.OrderNo]
		switch {
		case !ok:
			out = append(out, Mismatch{OnlyLocal, l, nil})
		case *l != *r:
			out = append(out, Mismatch{Different, l, r})
		}
		delete(idx, l.OrderNo)
	}
	for _, r := range idx {
		out = append(out, Mismatch{OnlyRemote, nil, r})
	}
	return out
}
```

`Diff` 的过程就是前面说的逐层向下比较：每一层把所有“可疑”节点的编号一次性发给对端，拿回它们的哈希，只有不一致的节点才会展开到下一层。这样不管差异有多少，往返次数固定为 17 次（树高 16 再加根这一层），每次都是一个批量请求。

### 实测效果

我用 100 万条记录测试了一下，在对端数据中制造了 4 处差异：一条金额不一致、一条状态不一致、少一条记录、多一条记录，并且把对端记录的顺序整体打乱：

| 指标 | 结果 |
| --- | --- |
| 建两棵树的耗时 | 约 0.4 秒 |
| 定位不一致的桶 | 17 次往返，共获取 117 个哈希，耗时约 20 微秒（不含网络） |
| 需要从对端拉取的记录 | 63 条（全量是 100 万条） |
| 找到的差异 | 4 处，类型全部正确 |
| 两边完全一致时 | 比较一次根节点就结束 |

可以看到，**真正需要传输和逐条比较的数据量，只和差异的数量有关，而和总数据量几乎无关。** 设差异落在 $d$ 个桶里，树高为 $k$，需要获取的哈希数量最多约为 $2dk$ 个，需要逐条比较的记录约为 $d \cdot \frac{N}{2^k}$ 条。

---

## 增量维护：从 T+1 走向准实时

上面的实现是一次性建树。如果希望做到准实时对账，就需要在数据变化时**增量更新**这棵树：

1. 一条记录发生变化，算出它所在的桶；
2. 重新计算这个桶的叶子哈希；
3. 沿着到根的路径，重新计算 16 个内部节点的哈希。

第 2 步需要读出这个桶里的所有记录再排序计算。如果连这一步都想省掉，可以换一种叶子哈希的定义：**把桶内每条记录的哈希当作一个大整数，求和后对 $2^{256}$ 取模。** 加法满足交换律，所以与记录顺序无关；新增一条记录，就加上它的哈希；修改一条记录，就减去旧哈希、加上新哈希，完全不需要读取桶内的其他记录。

千万**不要用异或**代替求和。异或虽然也满足交换律，但它有一个致命的性质：$a \oplus a = 0$。同一条记录如果被重复入账了两次，两份哈希会互相抵消，就好像这条记录不存在一样，而重复入账恰恰是对账最需要发现的问题之一。

---

## 跨日与在途：对账里最麻烦的部分

理论上很美好，但实际对账中，大量的“差异”其实并不是真正的错误，而是**时间边界**造成的。

比如一笔支付在 23:59:59 发起，本地记为当天成功，渠道却在 00:00:01 才完成处理，记到了第二天的账单里。在当天的对账结果中，它会表现为“本地有、对端无”；第二天又会表现为“对端有、本地无”。

常见的处理方式是：

- **先挂账，不要急着报错**：差异记录先放入一个“待确认”池；
- **跨日复核**：拿这些记录去和前后一天的树（或桶）再比较一次，大部分跨日差异会在这一步自动消除；
- **多次对账**：T+1 做初次对账，T+2 再复核一次，仍然对不上的才进入差错处理流程。

退款也有类似的“在途”问题：退款发起了，但渠道还没处理完，两边的状态会有一段时间不一致。

---

## 差错处理

真正的差错被确认之后，按类型处理：

- **对端有、本地无（掉单）**：调用渠道的查单接口确认结果，然后**补单**，把本地的支付单更新为成功，并触发后续的发货等流程。补单和正常的支付回调走的是同一套逻辑，所以必须是幂等的，否则补单和迟到的回调会导致重复发货；
- **本地有、对端无**：同样先查单确认。如果渠道确实没有成功，就需要冲正本地状态，必要时拦截发货；
- **金额或状态不一致**：通常需要人工介入，排查是退款未同步、手续费口径不同，还是代码 bug。

所有的差错及其处理过程都应该记录下来，形成可追溯的差错台账。

---

## Merkle Tree 不是银弹

最后也要说清楚它的适用边界：

- **两边都能建树，才能省掉传输**：Merkle Tree 能减少传输量，前提是对端也按同样的规则建好了树，并且提供按节点查询哈希的接口。这在**内部系统之间**、跨机房的副本之间，或者和愿意配合的合作方之间是可行的；
- **对接第三方渠道时，账单还是要全量下载**：支付渠道只会给你一份对账单文件，不会帮你建树。这时 Merkle Tree 省不掉下载，但仍然有价值：账单下载后在本地建树，和本地数据的树比较，差错复核、数据补发后的重新对账都只需要比较变化的桶，每天的根哈希也可以存档，作为这一天账单的“指纹”；
- **规范化是最大的工作量**：树的部分其实很简单，真正花时间的是让两边对“同一条记录”算出同一个哈希：字段口径、金额单位、状态映射、时区和日切，任何一处不一致，都会让大量记录被误报为差异；
- **数据量不大就别过度设计**：每天几万条数据，一条 SQL 全量比较几秒钟就跑完了，没必要引入一套额外的机制。

---

## 小结

基于 Merkle Tree 的对账，核心思想是**利用差异的稀疏性**：先用哈希快速确认哪些部分一致并直接跳过，只把力气花在可能不一致的地方。

落地时要记住几点：

1. **按主键哈希分桶**，让树的形状只由主键决定，避免一条记录的增减导致整棵树错位；
2. **按对账维度和对账日分树**，每天的根哈希存档备查；
3. **严格规范化**：只取必须一致的字段，金额用整数，状态统一映射，编码无歧义；
4. **需要增量更新时，叶子用模加求和，不要用异或**；
5. **时间边界造成的差异先挂账、再复核**，确认后的差错按类型处理，补单必须幂等。

它不是要替代传统的全量对账，而是在数据规模足够大、差异足够少的时候，提供一种更省、更快的选择。
