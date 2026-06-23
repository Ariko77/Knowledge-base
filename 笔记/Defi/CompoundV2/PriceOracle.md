# PriceOracle

为 Comptroller 提供资产价格（各 cToken 的 underlying 价格），用来把不同资产折算到同一计价单位，进而计算用户的抵押价值、借款价值、清算门槛等

**常见实现**

* `PriceOracle` 接口（`getUnderlyingPrice(address cToken)`）是 Comptroller 调用的源头。历史上 Compound V2 用过 UniswapAnchoredView（UAV）+ Chainlink 的混合方案，后来有提案改为直接使用 Chainlink feeds。查看 docs 和 CIP（治理提案）能看到演进原因。[docs.compound.finance+1](https://docs.compound.finance/v2/prices/?utm_source=chatgpt.com)

**你看源码时关注**

* `getUnderlyingPrice` 的返回单位（通常是 1e(36 - underlyingDecimals) 或者 mantissa 标准，文档会说明）— 错误的单位会导致严重的借款/清算误判。


> 更新: 2025-11-03 20:42:13  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/bngrgq9zolv6grgy>