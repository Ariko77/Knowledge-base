# compound

1. 看白皮书
2. 看源码
3. 实现
4. 金融概念



借贷，

清算，抵押：

质押：给银行钱，吃利息

对冲：两边买的不一样

 

#  Compound V2 学习路线图（系统化）
## 阶段 1：宏观理解
目标：先知道协议的整体框架，明白玩家（借款人、存款人、清算人）的角色。  
● ✅ 阅读官方文档：Compound V2 Docs  
● ✅ 弄清楚 3 大核心合约：  
  ○ Comptroller：风险管理 + 市场管理  
  ○ cToken（cETH / cDAI …）：存款、借款、利息计算  
  ○ PriceOracle：资产价格  
● ✅ 搞懂角色关系：  
  ○ 供应者（Supplier） → 存款赚利息  
  ○ 借款人（Borrower） → 抵押资产借钱  
  ○ 清算人（Liquidator） → 平衡风险，赚取清算奖励

## 阶段 2：cToken 机制
目标：理解 cToken 的金融逻辑 + 代码实现。  
● 🔹 供应端逻辑  
  ○ mint() → 存入资产，获得 cToken  
  ○ redeem() / redeemUnderlying() → 赎回资产  
  ○ exchangeRate → 存款利息如何累计  
● 🔹 借款端逻辑  
  ○ borrow() → 借出资产，生成债务  
  ○ repayBorrow() / repayBorrowBehalf() → 偿还债务  
  ○ borrowBalanceStored() / borrowBalanceCurrent() → 借款利息计算  
👉 建议逐行看 cToken.sol（尤其是 mintFresh, redeemFresh, borrowFresh, repayBorrowFresh）。

## 阶段 3：借贷流程拆解
目标：把一次借贷的全链路吃透。

1. 存款 → 获得 cToken
2. enterMarkets() → 把某资产设为抵押
3. borrow() → 借出目标资产
4. 利息累积（accrueInterest）
5. repay → 清偿债务
6. redeem → 赎回抵押品  
👉 可以自己在测试网跑一遍：  
● 先 supply ETH，  
● 再 enterMarkets([cETH])，  
● 借出 DAI，  
● 然后尝试偿还和赎回。

# 阶段 4：风险与清算
目标：搞懂 Compound 如何保证系统安全。  
● 🔹 抵押因子（Collateral Factor）：决定你最多能借多少。  
● 🔹 健康度（Account Liquidity）：通过 Comptroller 计算。  
● 🔹 清算机制  
  ○ 借款超限 → 触发清算  
  ○ liquidateBorrow()：清算人帮还债，拿到抵押品（折扣价）  
👉 重点阅读 Comptroller 的 liquidateCalculateSeizeTokens。



# 阶段 5：利率模型
目标：理解借贷利率的动态调整。  
● JumpRateModel：常见利率模型  
  ○ 借贷利用率低时 → 利率低  
  ○ 借贷利用率高时 → 利率陡增（防止流动性枯竭）  
● 你需要会算：  
  ○ borrowRatePerBlock  
  ○ supplyRatePerBlock  
● 然后联系 exchangeRate，理解资金池如何自动调节。

# 阶段 6：实战演练
目标：动手写代码 & 调试交易，形成直觉。  
● ✅ 在 本地 fork 主网（Hardhat/Foundry），模拟借贷过程。  
● ✅ 写一个脚本，执行以下操作：  
  a. 存 1 ETH 到 cETH  
  b. 借 100 DAI  
  c. 打印 accountLiquidity  
  d. 等待一段时间，查看利息增长  
  e. 偿还借款，赎回抵押  
👉 这样你会对 “借款增长” 和 “健康度下降” 有实际感受。

# 阶段 7：进阶研究
目标：进入更深的协议层，方便后面学习安全性 & 扩展性。  
● 研究 治理机制（GovernorAlpha + GovernorBravo）  
● 对比 Compound V3（Comet） 与 V2 的设计差异  
● 阅读一些真实的 清算案例 / 攻击事件分析（比如 oracle 价格操纵、清算套利）





🚀 最终学习路径（提炼版）

1. 理解协议架构（Comptroller / cToken / Oracle）
2. 研究 cToken 源码（mint / redeem / borrow / repay）
3. 梳理借贷流程（从存款 → 借款 → 偿还 → 赎回）
4. 掌握风险控制（抵押因子 / 健康度 / 清算逻辑）
5. 理解利率模型（JumpRateModel）
6. 实战演练（本地 fork 主网 + 写交互脚本）
7. 进阶研究（治理、Compound V3、攻击案例）

要不要我帮你把 每个阶段的关键函数（比如 mintFresh、redeemFresh、seizeInternal 等）整理成一个 学习 checklist？这样你学完可以逐项打勾。



> 更新: 2025-10-28 19:52:29  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ymau8cacy2cagfwl>