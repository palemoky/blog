---
title: 为什么你的 Claude 周额度和别人不在同一天重置：从分布式系统看限额调度
date: 2026-11-06
draft: false
tags: [AI, Claude, Distributed Systems]
---

如果你和身边用 Claude 的人对一下周额度的重置时间，大概率会发现：有人是周五，有人是周二，有人是周六凌晨。同样是 Max 订阅，重置时间却各不相同——既不是按订阅日，也不是统一的周一零点。

我们先看看官方规则，再从分布式系统的角度聊聊：**为什么一个服务海量用户的系统，几乎一定会把“重置时间”打散。**

---

## 官方规则：每个账号一个固定的重置时刻

目前 Pro 和 Max 计划有两层限额：

- **5 小时会话限额**：滚动窗口，从你开始使用起算；
- **周限额**：每周重置一次，Max / Ultra 还另有一个单独的 Fable 周限额。

关于周限额何时重置，Anthropic 帮助中心的 [What is the Max plan](https://support.claude.com/en/articles/11049741-what-is-the-max-plan) 写得很明确：

> The weekly limit resets at a fixed time each week that is assigned to your account.

也就是说：

1. 重置时间是**分配给账号的**，不是你自己选的；
2. 它**不随你何时开始使用、何时订阅而变化**；
3. 每个周期都会拿到完整的周额度。

自己的重置时间可以在 claude.ai 的 **Settings → Usage** 查看，Claude Code 里用 `/usage` 也能看到。

---

## 如果所有人都在周一0点重置，会发生什么？

想象一个反例：所有用户的周额度都在 UTC 周一 00:00 统一重置。

这在分布式系统里有个经典的名字——**惊群效应（Thundering Herd）**。

- 周日晚上，大量重度用户额度已经见底，只能干等；
- 零点一过，所有人同时“满血复活”，积压的任务一起涌进来；
- 推理集群在这一刻面对的是平时数倍的突发流量，而 GPU 容量是按峰值采购的，这个峰值每周只出现一次，其余时间大量闲置。

同样的问题在其他场景里也很常见：

| 场景 | 惊群表现 | 常见解法 |
| --- | --- | --- |
| 缓存同时过期 | 大量请求同时击穿到数据库 | 过期时间加随机抖动 |
| 客户端失败重试 | 服务恢复瞬间被重试打垮 | 指数退避 + Jitter |
| Cron 任务都配成 `0 0 * * *` | 整点 CPU 尖刺 | 错峰调度、随机延迟 |
| 证书/Token 同一时刻批量签发 | 到期时集中续签 | 有效期打散 |

**周额度重置也是同一个问题**，解法自然也一样：把重置时刻打散到一周 168 个小时里。

---

## 打散重置时间的一种可能实现

Anthropic 并没有公开具体如何分配重置时间，以下是我按常见做法推测的一种实现，仅用于说明思路：

```python
import hashlib

WEEK_SECONDS = 7 * 24 * 3600

def reset_offset(account_id: str) -> int:
    """根据账号 ID 计算其在一周内的固定重置偏移（秒）。"""
    # 返回一个 bytes（原始二进制字节串）类型，长度固定为 32 个字节（256 位）
    digest = hashlib.sha256(account_id.encode()).digest()
    # 截取 8 字节是为了将其转为一个 64 位整数，不仅精度足够满足均匀分布，计算效率也极高
    # 通过大端字节序（Big-Endian）直接将二进制字节转换为整数
    return int.from_bytes(digest[:8], "big") % WEEK_SECONDS

def next_reset(account_id: str, now: int, epoch: int = 0) -> int:
    """返回 now 之后该账号的下一次重置时间戳。"""
    offset = reset_offset(account_id)
    elapsed = (now - epoch - offset) % WEEK_SECONDS
    return now + (WEEK_SECONDS - elapsed)
```

这个设计有几个好处：

1. **确定性**：同一个账号算出来的结果永远一样，不需要额外存储，任何一个网关节点都能独立算出，天然适合分布式部署；
2. **均匀性**：哈希结果近似均匀分布，海量账号会被平铺到一周的每一个小时，重置带来的流量回升变成一条平滑的曲线，而不是一根尖刺；
3. **与使用行为解耦**：不依赖“第一次使用时间”，用户无法通过刻意卡点来操纵窗口，也就不会出现“所有人都在同一时间开始用”导致的二次聚集。

当然，实际系统也可能是在账号创建时随机生成一个偏移量并持久化，效果是一样的。

---

## 被打散的代价：全局重置的“不公平”

打散重置时间对系统很友好，但也带来一个有趣的副作用。

2026 年 7 月，Anthropic 连续三周（7 月 1 日、9 日、16 日）宣布[为所有用户重置 5 小时和周限额](https://www.implicator.ai/anthropic-resets-claude-limits-for-third-week-without-explaining-why/)。关键在于：**全局重置只清空了用量，并没有改变每个账号原本分配的重置时刻。**

于是不同账号拿到的“额外额度”差别很大。以 7 月 16 日（周四）这次为例，假设用户匀速使用，全局重置相当于把“自上次常规重置以来已经用掉的部分”还给你，额外收益约为 $\dfrac{d}{7}$，其中 $d$ 是距上次常规重置已过去的天数：

| 账号常规重置日 | 距上次常规重置已过去 | 约白赚的周额度 |
| --- | --- | --- |
| 周五 | 约 6 天（上周五） | $6/7 \approx 86\%$ |
| 周日 | 约 4 天（上周日） | $4/7 \approx 57\%$ |
| 周四 | 不到 1 天（当天刚重置） | ≈ 0% |

这组数字来自 Reddit 用户的估算，非官方数据。

有 Reddit 用户收集了 49 个时区、196 名用户的重置时间做分析（见 [ProPakistani 的报道](https://propakistani.pk/2026/07/28/claude-users-are-getting-free-usage-limit-extensions-heres-how/)），结论是在全局重置中，“最赚”和“最亏”的重置窗口之间收益差距约 **2.7 倍**。样本量不大、且是自愿上报，看个热闹即可，但它从侧面印证了：**重置时间确实是被打散在一周之中的。**

这其实是分布式系统里另一个常见的权衡：**为了系统整体平稳，个体之间必然存在差异**。就像缓存过期加了 Jitter 之后，每个 key 的实际存活时间都不一样；全局性的“补偿”操作落到每个个体上，效果也必然不均匀。

---

## 对我们用户的启示

1. **先查清自己的重置时间**：Settings → Usage 或 `/usage`，别凭感觉；
2. **按自己的周期规划重活**：比如重置前一天额度还剩很多，可以安排大型重构或批量任务，不用就浪费了；
3. **全局重置不是人人等价**：如果恰好在自己常规重置前几小时遇到全局重置，收益几乎为零，这不是 bug，而是打散设计的自然结果；
4. **遇到重置日漂移**：可以截图留证后通过官方渠道反馈，因为按目前帮助中心的说法，重置时刻应该是固定的。

---

## 小结

“每个人的周额度在不同时间重置”看起来只是个产品细节，背后其实是分布式系统里最朴素的一条原则：**不要让所有人在同一时刻做同一件事。** 缓存过期的 Jitter、重试的指数退避、Cron 的错峰调度，以及 Claude 的周额度重置，本质上都是同一个思路——用确定性的“打散”，换取整个系统负载的平滑。

---

**参考资料**

- [What is the Max plan? - Claude Help Center](https://support.claude.com/en/articles/11049741-what-is-the-max-plan)
- [Using Claude Code with your Pro or Max plan - Claude Help Center](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
- [anthropics/claude-code#54974：周重置日漂移问题](https://github.com/anthropics/claude-code/issues/54974)
- [Anthropic resets Claude limits for third week without explaining why - Implicator](https://www.implicator.ai/anthropic-resets-claude-limits-for-third-week-without-explaining-why/)
- [Claude Users Are Getting Free Usage Limit Extensions — ProPakistani](https://propakistani.pk/2026/07/28/claude-users-are-getting-free-usage-limit-extensions-heres-how/)
