# 查漏补缺

# 写一些 Foundry 测试脚本去验证下列场景（每条都可写成单元测试）：

1. **基本 swap 流程**：记录 `reserve` 在 swap 前后变化，验证 `x*y>=k`（含手续费 0.3%）
2. **提供流动性**：用 direct pair.transfer+mint 与 router.addLiquidity 两条路径对比结果（amounts / lpt）
3. **移除流动性**：burn 前后 verify token returned 按比例并且 lpt 减少
4. **flash swap 测试**：模拟 callee 在回调内做任意操作（借用并偿还）
5. **异常路径**：swap 时不转入 token，验证 `INSUFFICIENT_INPUT_AMOUNT` revert
6. **带手续费代币**：用一个模拟 token 在 transfer 中扣除 fee 测试 router/pair 的容错性
7. **TWAP**：模拟时间推进并记录 `priceCumulative`，用两次采样算出 TWAP 并验证值

pair怎么检验到用户添加或者减少资金的

我们在自己调用swap的时候怎么检查我们的

核心swap

rounter和pair的关系

* 抢跑


> 更新: 2025-10-17 20:10:55  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/llza61g8qznar7vt>