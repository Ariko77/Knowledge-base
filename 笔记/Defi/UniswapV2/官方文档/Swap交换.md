# Swap 交换

# introduce

在 Uniswap 中进行代币交换（Token swaps）是一种将一种 ERC-20 代币交易为另一种代币的简单方式。

对于终端用户来说，交换是直观的：用户选择一个输入代币和一个输出代币。他们指定一个输入数量，协议会计算他们将收到多少输出代币。然后，他们只需点击一次即可执行交换，并立即在他们的钱包中收到输出代币。

在本指南中，我们将研究在协议层面上交换时发生了什么，以便更深入地理解 Uniswap 的工作原理。

Uniswap 中的交换与传统平台上的交易不同。Uniswap 不使用订单簿来表示流动性或决定价格。Uniswap 使用自动化做市商机制 (automated market maker mechanism) 来即时反馈汇率和滑点。

正如我们在《协议概览》中所学到的，Uniswap 上的每一个交易对实际上都是由一个流动性池支撑的。流动性池是智能合约，它们持有两种独特代币的余额，并强制执行有关存入和提取这些代币的规则。

这种规则就是常数乘积公式 (constant product formula)。当任一代币被提取（购买）时，必须按比例存入（出售）另一种代币，以维持常数关系。

***

## 交换的剖析 (Anatomy of a swap)

在最基本的层面上，Uniswap V2 中所有的交换都发生在一个单一的函数中，这个函数的名字非常贴切，叫做 **swap**：

```solidity
function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data);
```

### 接收代币 (Receiving tokens)

从函数签名中应该可以清楚看出，Uniswap 要求交换的调用者通过 `amount{0,1}Out` 参数来指定他们希望接收多少输出代币，这些参数对应于所需的 `token{0,1}` 数量。

### 发送代币 (Sending Tokens)

不太明显的是，Uniswap 如何接收代币作为交换的支付。通常，需要使用代币的智能合约在执行某些功能之前，会要求调用者先在代币合约上进行 **approve** 授权，然后再调用一个函数，该函数进一步在代币合约上调用 `transferFrom`。这并不是 V2 交易对 (pairs) 接收代币的方式。

相反，交易对在每次交互结束时检查它们的代币余额。然后，在下一次交互开始时，当前余额与存储的值进行差分，以确定当前交互者发送了多少代币。有关为什么这样设计的理由，请参见白皮书。

需要记住的是：代币必须在调用 `swap` 之前先转移到交易对中（对此规则的唯一例外是闪电交换 Flash Swaps）。这意味着，为了安全地使用 `swap` 函数，它必须从另一个智能合约中调用。另一种方式（先将代币转到交易对，然后再调用 `swap`）在非原子执行时并不安全，因为已发送的代币会暴露在套利风险之下。

***

## 开发者资源 (Developer resources)

* 要查看如何在智能合约中实现代币交换，请阅读 **Trading from a smart contract**。
* 要查看如何从接口执行交换，请阅读 **Trading (SDK)**。


> 更新: 2025-09-22 21:01:04  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/gclx8o2ufm0aadgu>