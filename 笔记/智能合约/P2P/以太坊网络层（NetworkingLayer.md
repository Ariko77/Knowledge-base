# 以太坊网络层（Networking Layer

以太坊是一个由成千上万节点组成的点对点（P2P）网络。所有节点必须使用统一的协议才能互相发现、通信、同步区块与交易。所谓“网络层”，就是这些协议组合成的一整套通信机制，包括：

+ Gossip（八卦扩散）——一对多广播消息
+ Request / Response（请求 / 响应）——节点之间的一对一数据交换
+ 节点发现机制
+ 安全传输协议

并且，以太坊现在有两类客户端：

1. 执行层客户端（Execution Clients）  
管交易、EVM 执行、交易池、合约状态等。
2. 共识层客户端（Consensus Clients）  
管信标链、区块提议、验证者工作、Attestation 等。

两者各有自己的网络栈，还需要彼此通信。

# 一、执行层（Execution Layer）的网络栈
执行层的网络系统分为两个并行的部分：

1）Discovery：节点发现机制（基于 UDP）

允许新节点找到其他节点。

2）DevP2P：节点数据交换（基于 TCP）

完成真正的数据传输：交易同步、握手认证、子协议协作等。

这两块一起运行：

```plain
Discovery 帮你找到人
DevP2P 负责和人聊天
```

## （一）Discovery：节点发现机制
目标：让新节点加入网络并找到可连接的邻居。

1. 启动依赖 bootnodes（引导节点）

每个客户端都内置一些固定地址的 bootnodes。它们只负责：

接受新节点的“自我介绍”

回答：“你可以去找这些 peers”

bootnodes 不参与同步、不挖矿、不验证区块，只做“介绍人”。

2. 协议基础：Kademlia（DHT 分布式哈希表）

距离不是指地理距离，而是“节点 ID 的相似度”。

每个节点维护一张“路由表”，记录离自己最近 ID 的节点。

3. Discovery 的流程：PING–PONG bonding

新节点上线时，与 bootnode 进行：

PING → PONG（哈希验证） → Bonding（建立可信关系）

随后发起 FIND_NEIGHBOURS 请求获得更多节点。

流程图：

启动 → 找 bootnode → PING/PONG 建立关系 → 获取邻居列表 → 和邻居做 PING/PONG → 完整加入网络



目前执行层主要使用 Discv4，正在迁移至 Discv5。

## （二）ENR：Ethereum Node Record
ENR 是一种“节点身份证格式”，内容包括：

节点签名（根据身份方案生成）

序列号（记录更新次数）

多种键值对：公钥、IP、端口、支持的协议……

ENR 的设计是可扩展、向前兼容的，因此它成为推荐的节点标识格式。

## （三）为什么节点发现使用 UDP？
UDP：

无连接、无重传、无检测

超低开销，速度快

发现阶段只需要：

“我在这儿——你听到就好。”

正式通信才用 TCP，因为需要：

错误检查

重传

有序可靠

所以：

Discovery → UDP

DevP2P → TCP

# 二、DevP2P：执行层的正式通信协议栈
DevP2P 由多层协议组成：

+ RLPx（会话层 + 加密）
+ Wire Protocol（基础消息层）
+ 一系列子协议（同步、交易、轻节点等）

## （一）RLPx：安全握手与会话维护
RLPx 负责：

1. 建立连接
2. 密钥交换
3. 消息加密
4. 维持会话

过程：

1. 发出 auth 消息
2. 对方向你验证身份
3. 如果成功，对方返回 auth-ack
4. 完成密钥交换
5. 开始发送 “hello” 消息

hello 消息包含：

+ 协议版本
+ 客户端 ID
+ 端口
+ 节点 ID
+ 支持的子协议列表

双方比对各自支持的子协议，找出能一起使用的。

## （二）Wire Protocol（基础线协议）
包含：

+ 链同步（已交给共识层）
+ 区块传播（已交给共识层）
+ 交易传播（仍由执行层负责）

还有：

+ Disconnect 信息

定期 PING–PONG 保持会话不过期

## （三）主要子协议
1. les（light client 协议）

轻节点同步协议，但较少被使用，因为：

需要大量全节点无偿提供服务，因此默认关闭。

2. snap（快照同步协议）

允许节点交换最近状态的快照：

不需要完整下载中间的 Merkle Trie 节点

大幅加速节点同步

3. wit（witness 协议）

用于：

交换“状态见证（witness）”

帮节点更快同步到链顶端

4. whisper（消息通信协议，已弃用）

原本用于 P2P 安全通信，现在已废弃。

# 三、共识层（Consensus Layer）
共识层有自己独立的 P2P 网络规范：

+ 需要参与区块 Gossip
+ 需要接收 / 广播 Attestation
+ 需要验证 Beacon 区块
+ 使用 libp2p，不使用 DevP2P

流程类似执行层：

1. Discovery（UDP + Discv5）
2. 建立加密连接（Noise 协议）
3. Gossip（广播区块、证明、惩罚事件）
4. Request/Response（请求特定区块）

## （一）libP2P 通信
libP2P 分为两类协议：

1）Gossip（Gossipsub v1）

用于需要快速传播的内容：

+ 信标区块
+ Attestations
+ Slashings
+ Exits

每个节点会限制消息大小、缓存记录等。

2）Req/Resp（请求响应）

用于精确请求：

+ 请求某个 Beacon 区块
+ 请求某段 slot 范围的块

响应内容：

+ 使用 Snappy 压缩
+ 使用 SSZ（Simple Serialization）编码

## （二）为什么共识层使用 SSZ 而不是 RLP？
SSZ 的优势：

1. 有固定偏移量，取字段不用解整个结构
2. 为 Merkleization 优化（SSZ 的编码直接适配 Merkle root）
3. 所有值有唯一表示（RLP 存在多种表示方式）

适合：

1. 验证者处理大量 Merkle 数据
2. 执行轻量快速的结构解码

# 四、执行层与共识层如何连接（Engine API）
执行层（EL）与共识层（CL）并行运行，需要通过本地 RPC互相通信。

例如：

当不是区块提议者时（普通验证流程）

CL 接收新块（共识 P2P）

CL 先检查块是否来自正确的发送者

CL 把执行载荷发送给 EL（本地 RPC）

EL 执行交易，验证头部状态哈希

EL 把验证结果返回 CL

CL 确认区块有效，广播 Attestation（共识 P2P）

当本节点是区块提议者时（出块流程）

CL 得到“你是下一个提议者”的通知

CL 调用 EL 的 create_block（本地 RPC）

EL 从交易池（执行层 P2P）取交易

EL 打包执行交易，生成执行载荷

CL 获取载荷，封装成 Beacon block

CL 广播新块（共识 P2P）

其他节点验证并最终确认区块

## 总结版：一句话记住整个网络层
执行层（DevP2P）负责交易广播，共识层（libP2P）负责区块 Gossip。  
两者使用不同的网络栈，通过 Engine API 协作完成出块与链同步。



> 更新: 2025-11-14 11:29:49  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/gzul9wlzhzxskiqi>