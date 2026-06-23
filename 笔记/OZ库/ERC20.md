# ERC20

# ERC20

`IERC20`是`ERC20`代币标准的接口合约，规定了`ERC20`代币需要实现的函数和事件

```solidity
pragma solidity ^0.8.20;

interface IERC20 {
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 value) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 value) external returns (bool);
    function transferFrom(address from, address to, uint256 value) external returns (bool);
}
```

## 6个函数

1. `totalSupply()`：查看此合约发行的ERC-20代币总量，![](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\image-20250508200805691.png)
2. `balanceOf() `：某个地址上的余额![](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\image-20250508200818847.png)
3. `transfer() `： 发送token，即转账

![](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\image-20250508200842666.png)

```
- to不能是一个空地址
- caller至少有value这么多余额
- emit一个transfer事件
```

4\. `allowance() `：查看owner给spender授权的可转账额度![](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\image-20250508200859979.png)
5\. `approve()` ：owner给spender授权一定的转账额度（来自owner账户）

![](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\image-20250508200912125.png)

```
- spender不能是一个空地址
- emit一个approval事件
```

6\. `transferFrom()`： spender用owner的代币进行转账，金额需要同时小于allowance和owner的余额
\- from和to都不可以是空地址
\- from的余额至少有value这么多
\- caller必须有来自from的至少value这么多的额度(allowance),就是from需要提前为msg.sender授权，即 from亲自去调用approve()函数

## 2个事件

触发事件需要<font style="background-color:#f3bb2f;">emit</font>，不是条件成立就直接触发

1. `Transfer()` ： **转账**成功触发的事件触发条件：当 `value` 单位的货币从账户 (`from`) 转账到另一账户 (`to`)时![](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\image-20250508194605949.png)
2. `Approval()` ：给某账户**授权**一定额度的事件触发条件：当 `value` 单位的货币从账户 (`owner`) 授权给另一账户 (`spender`)时![](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\image-20250508194526435.png)

### \_mint() 和 \_burn()

都是internal类型

* \_mint() :

```solidity
_mint(address to, uint256 amount)
```

给地址 `to` 铸造（发放）`amount` 份额

* \_burn():

```solidity
_burn(address from, uint256 amount)
```

从地址 `from` 销毁 `amount` 份额

# **ERC20**

`ERC`代表`Ethereum Request for Comment`。从本质上讲，它们是已获得社区批准的标准，用于传达某些用例的技术要求和规范。

`ERC-20`特别是一个标准，它概述了可替代代币的技术规范.

## 1. 6个函数

* `totalSupply()`： token的总量
* `balanceOf() `：某个地址上的余额
* `transfer() `： 发送token
* `allowance() `：额度、配额、津贴
* `approve()` ： 批准给某个地址一定数量的token(授予额度、授予津贴)
* `transferFrom()`： 提取approve授予的token(提取额度、提取津贴)

| 标准函数 | 含义 |
| --- | --- |
| **totalSupply()** | 代币总量 |
| **balanceOf**(addresss account) | account地址上的余额 |
| **transfer**(address recipient, uint256 amount) | 向recipient发送amount个代币 |
| **allowance**(address owner, address spender) | 查询owner给spender的额度(总配额) |
| **approve**(address spender, uint256 amount) | 批准给spender的额度为amount(当前配额) |
| **transferFrom**(address sender, address recipient, uint256 amount) | recipient提取sender给自己的额度 |

## 2. 2个事件

* `Transfer()` ： token转移事件
* `Approval()` ：额度批准事件

| **事件** | **含义** |
| --- | --- |
| **Transfer**(address indexed from, address indexed to, uint256 value) | 代币转移事件：从from到to转移value个代币 |
| **Approval**(address indexed owner, address indexed spender, uint256 value) | 额度批准事件：owner给spender的额度为value |

## 3. 实践(还没学完)

参考的网址<https://learnblockchain.cn/article/4327(不知道靠不靠谱)>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol";//这都是一行

    contract LW3Token is ERC20 {
    constructor(string memory _name, string memory _symbol) ERC20(_name, _symbol) {
    _mint(msg.sender, 10 * 10 ** 18);
    }
}
```

在构造函数中，我们需要来自用户的两个参数——`_name`它们`_symbol`指定我们加密货币的名称和符号。例如。名称 = 以太坊，符号 = ETH

在指定构造函数后，我们立即调用`ERC20(_name, _symbol)`

`ERC20`我们从 OpenZeppelin 导入的合约有它自己的构造函数，它需要`name`和`symbol`参数。由于我们正在扩展 ERC20 合约，因此我们需要在部署 ERC20 合约时**初始化 ERC20 合约**。

所以，作为我们构造函数的一部分，我们还需要**调用**`ERC20`**合约上的构造函数**。因此，我们为我们的合约提供`_name`和`_symbol`变量，我们立即将其传递给`ERC20`构造函数，从而初始化ERC20智能合约。\`

`_mint`是标准合约中的一个`internal`函数`ERC20`,`_mint`接受两个参数 - 铸造地址和铸造代币数量

调用该`_mint`函数将一些代币铸造到`msg.sender`

`10 * 10 ** 18`:我希望将 10 个完整令牌铸造到我的地址


> 更新: 2025-07-11 11:22:26  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/myi1fhy4g40750w5>