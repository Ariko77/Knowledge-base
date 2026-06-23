# ERC4626

| 英文原词 |  |
| --- | --- |
| Vault | 金库 / Vault |
| shares | 份额（share） |
| value | 实际资产数量 |
| baseUnit | 精度单位（如 1e18） |
| exchangeRate | 份额兑换率 |
| holdings | 金库当前总资产 |

<font style="color:rgb(51, 51, 51);">ERC4626 是一个代币化保险库标准，它使用 ERC20 代币来表示某种其他资产的股份。</font>

<font style="color:rgb(51, 51, 51);">它的工作原理是，你将一种 ERC20 代币（代币A）存入 ERC4626 合约，然后获得另一种 ERC20 代币，称为代币 S。</font>

<font style="color:rgb(51, 51, 51);">在这个例子中，</font>**<font style="color:rgb(51, 51, 51);">代币 S 代表你在合约中所拥有的所有代币 A 的股份</font>**<font style="color:rgb(51, 51, 51);">（而不是代币 A 的总供应量，仅限于 ERC4626 合约中的代币 A 余额）。</font>

<font style="color:rgb(51, 51, 51);">在稍后的时间里，你可以将代币 S 放回保险库合约，获得代币 A 的返还。</font>

<font style="color:rgb(51, 51, 51);">如果保险库中代币 A 的余额增长速度快于代币 S 的发行速度，你将按比例提取比你存入的代币 A 更多的代币 A。</font>

当 ERC4626 合约给你一个 ERC20 代币作为初始存款时，它给你的是代币 S（一个符合 ERC20 标准的代币）。这个 ERC20 代币并不是一个单独的合约，而是实现于 ERC4626 合约中。实际上，你可以看到 OpenZeppelin 在 Solidity 中是如何定义这个合约的：

```solidity
abstract contract ERC4626 is ERC20, IERC4626 {
    using Math for uint256;

  
    IERC20 private immutable _asset;
    uint8 private immutable _underlyingDecimals;

    /**
     * @dev Set the underlying asset contract. This must be an ERC20-compatible contract (ERC20 or ERC777).
     */
    constructor(IERC20 asset_) {
        (bool success, uint8 assetDecimals) = _tryGetAssetDecimals(asset_);
        _underlyingDecimals = success ? assetDecimals : 18;
        _asset = asset_;
    }

```

<font style="color:rgb(51, 51, 51);">ERC4626 扩展了 ERC20 合约，并在构造阶段接受作为参数的其他 ERC20 代币，用户将把其存入该合约。</font>

<font style="color:rgb(51, 51, 51);">这个代币被称为 ERC4626 中的股份。它就是 ERC4626 合约本身。</font>

<font style="color:rgb(51, 51, 51);">你拥有的股份越多，你对存入的基础资产（其他 ERC20 代币）就拥有越多的权利。</font>

<font style="color:rgb(51, 51, 51);">每个 ERC4626 合约只支持一种资产。你不能将多种 ERC20 代币存入合约并获得股份</font>

# <font style="color:rgb(51, 51, 51);">动机</font>

<font style="color:rgb(51, 51, 51);">让我们用一个真实的例子来解释这个设计。</font>

<font style="color:rgb(51, 51, 51);">假设我们都拥有一个公司或一个流动性池，定期赚取稳定币 DAI。在这种情况下，稳定币 DAI 是资产。</font>

<font style="color:rgb(51, 51, 51);">一种低效的方式是按比例将 DAI 分发给每个公司持有者。但从 gas 费的角度来看，这将非常昂贵。</font>

<font style="color:rgb(51, 51, 51);">同样，如果我们要在智能合约内更新每个人的余额，这也将很昂贵。</font>

<font style="color:rgb(51, 51, 51);">相反，使用 ERC4626，工作流程将是这样的。</font>

<font style="color:rgb(51, 51, 51);">假设你和九个朋友聚在一起，每个人各存入 10 DAI 到 ERC4626 保险库（总共 100 DAI）。你将获得一个股份。</font>

<font style="color:rgb(51, 51, 51);">到目前为止一切都很好。现在你们的公司赚取了 10 个 DAI，因此保险库中的总 DAI 现在为 110 DAI。</font>

<font style="color:rgb(51, 51, 51);">当你将你的股份兑换回你的 DAI 时，你不会得到 10 DAI，而是 11 DAI。</font>

<font style="color:rgb(51, 51, 51);">现在保险库中有 99 DAI，但有 9 个人需要分享。如果他们每个人都提取，他们将各得到 11 DAI。</font>

<font style="color:rgb(51, 51, 51);">请注意这有多高效。当有人进行交易时，合约中只需更改股份的总供应量和资产的数量，而不是逐个更新每个人的股份。</font>

<font style="color:rgb(51, 51, 51);">ERC4626 不必以这种方式使用。你可以使用任意数学公式来确定股份与资产之间的关系。例如，你可以规定每次有人提取资产时，他们还必须支付某种依赖于区块时间戳的税费或类似的费用。</font>

<font style="color:rgb(51, 51, 51);">ERC4626 标准提供了一种节省 gas 费的方式，用于执行非常常见的 DeFi 实践。</font>

# <font style="color:rgb(51, 51, 51);">概念</font>

## 股份

### <font style="color:rgb(255, 80, 44);background-color:rgb(255, 245, 245);"></font><code><font style="color:rgb(255, 80, 44);background-color:rgb(255, 245, 245);">asset</font></code><font style="color:rgb(51, 51, 51);"> 函数</font>

<font style="color:rgb(51, 51, 51);">返回用于保险库的基础代币的地址</font>：

```solidity
function asset() returns (address)
```

<font style="color:rgb(51, 51, 51);">比如如果基础资产是 DAI，那么该函数将返回 DAI 的 ERC20 合约地址 </font><code><font style="color:rgb(255, 80, 44);background-color:rgb(255, 245, 245);">0x6b175474e89094c44da98b954eedeac495271d0f</font></code>

### <code><font style="color:rgb(255, 80, 44);background-color:rgb(255, 245, 245);">totalAssets</font></code><font style="color:rgb(51, 51, 51);"> 函数</font>

<font style="color:rgb(51, 51, 51);">返回保险库“管理”（拥有）的资产总量，即 ERC4626 合约拥有的 ERC20 代币数量：</font>

```solidity
function totalAssets() returns (uint256)
```

<font style="color:rgb(51, 51, 51);">OpenZeppelin 中的实现非常简单：</font>

```solidity
/** @dev See {IERC4626-totalAssets}. */
function totalAssets() public view virtual override returns (uint256) {
  return _asset.balanceOf(address(this));
}
```

<font style="color:rgb(51, 51, 51);">当然，没有获取“股份地址”的函数，因为那就是 ERC4626 合约的地址。</font>

# methods

#### deposit 存钱,给股

```solidity
function deposit(address _to, uint256 _value) public returns (uint256 _shares)
```

<font style="color:rgb(34, 34, 34);">将</font><code><font style="color:rgb(34, 34, 34);">_value</font></code><font style="color:rgb(34, 34, 34);">代币存入保险库并将其所有权授予</font><code><font style="color:rgb(34, 34, 34);">_to</font></code><font style="color:rgb(34, 34, 34);">。</font>

<code><font style="color:rgb(34, 34, 34);">_shares</font></code><font style="color:rgb(34, 34, 34);">可以返回相应的按比例所有权值</font><code><font style="color:rgb(34, 34, 34);">_value</font></code><font style="color:rgb(34, 34, 34);">，如果不是，则必须返回</font><code><font style="color:rgb(34, 34, 34);">0</font></code>

#### <font style="color:rgb(34, 34, 34);">redeem 给股,分钱</font>

```solidity
function redeem(address _to, uint256 _shares) public returns (uint256 _value)
```

赎回指定数量的份额（`_shares要赎回的份额数量`），Vault 将根据当前兑换率把相应数量的原始资产（`_value实际收到的资产数量`）转给 `_to`\
如果不能精确计算资产金额，也必须返回`0`表示无效

#### <font style="color:rgb(34, 34, 34);">withdraw 给钱,退股</font>

```solidity
function withdraw(address _to, uint256 _value) public returns (uint256 _shares)
```

<code><font style="color:rgb(34, 34, 34);">_value</font></code><font style="color:rgb(34, 34, 34);">从保险库中</font><font style="color:rgb(34, 34, 34);">取出代币并将其转移到</font><code><font style="color:rgb(34, 34, 34);">_to</font></code><font style="color:rgb(34, 34, 34);">。</font>

<code><font style="color:rgb(34, 34, 34);">_shares</font></code><font style="color:rgb(34, 34, 34);">可以返回对应于的按比例所有权值</font><code><font style="color:rgb(34, 34, 34);">_value</font></code><font style="color:rgb(34, 34, 34);">，如果不是，则必须返回</font><code><font style="color:rgb(34, 34, 34);">0</font>**<font style="color:rgb(34, 34, 34);"></font>**</code>

* `_value`: 想取出的资产数量,单位是LING/DAI等代币
* `_shares` :  为了取出 `_value` 资产，你**销毁的份额数量**

#### <font style="color:rgb(34, 34, 34);">totalHoldings</font>

```solidity
function totalHoldings() public view returns (uint256)
```

<font style="color:rgb(34, 34, 34);">返回保险库持有/管理的底层代币总量</font>

#### <font style="color:rgb(34, 34, 34);">balanceOfUnderlying</font>

```solidity
function balanceOfUnderlying(address _owner) public view returns (uint256)
```

<font style="color:rgb(34, 34, 34);">返回保险库中持有的底层代币总量</font><code><font style="color:rgb(34, 34, 34);">_owner</font></code>

#### <font style="color:rgb(34, 34, 34);">underlying</font>

```solidity
function underlying() public view returns (address)
```

<font style="color:rgb(34, 34, 34);">返回保险库用于记账、存款和取款的代币地址</font>

#### <font style="color:rgb(34, 34, 34);">totalSupply</font>

```solidity
function totalSupply() public view returns (uint256)
```

返回 Vault 中所有尚未赎回的总份额数量

#### <font style="color:rgb(34, 34, 34);">balanceOf</font>

```solidity
function balanceOf(address _owner) public view returns (uint256)
```

返回某个用户拥有的份额数量

#### <font style="color:rgb(34, 34, 34);">exchangeRate </font>

```solidity
function exchangeRate() public view returns (uint256)
```

获取当前每份 share 对应的资产数量（即兑换率), 也就是1 份 share 对应多少资产\
换算公式为：\
`资产数量 = shares × exchangeRate() ÷ baseUnit()`\
并且应满足：\
`exchangeRate() × 总份额（totalSupply） = Vault 总资产（totalHoldings`

这是核心比率函数,通胀攻击就会影响这个数值

#### <font style="color:rgb(34, 34, 34);">baseUnit</font>

```solidity
function baseUnit() public view returns(uint256)
```

返回 Vault 份额（shares）使用的单位基准值，通常是 `10 ** decimals()`。\
用来统一换算中涉及的精度。例如：

```solidity
assets = shares × exchangeRate / baseUnit
```

注意不是资产单位,而是用于兑换计算的单位比例

# Events

### <font style="color:rgb(34, 34, 34);">Deposit</font><font style="color:rgb(34, 34, 34);"> </font>

<font style="color:rgb(34, 34, 34);">当用户向 Vault 存入资产时触发该事件</font>

<font style="color:rgb(34, 34, 34);background-color:rgb(242, 242, 242);">event Deposit(address indexed \_from, addres indexed \_to, uint256 \_value)</font>

* `_from`：触发存入操作、批准资产的地址
* `_to`：获得份额的地址（可能与 `_from` 相同）
* `_vaule`: 表示存入的资产数量

### <font style="color:rgb(34, 34, 34);">Withdraw</font>

<font style="color:rgb(34, 34, 34);">当用户从 Vault 中提取资产时触发该事件  </font>

<font style="color:rgb(34, 34, 34);background-color:rgb(242, 242, 242);">event Withdraw(address indexed \_owner, addres indexed \_to, uint256 \_value)</font>

* `_owner`：份额的拥有者（用来换资产的）
* `_to`：接收资产的一方（通常是用户自己）
* `_vaule`: 表示赎回的资产数量

# 通货膨胀

简单来说就是攻击者先铸造极少份额,然后绕过share逻辑直接转入资产,让每个份额的价值暴涨,让后来的用户几乎买不到份额或者只能买到极少的份额,而攻击者能赎回整个vault的资产,白嫖别人钱


> 更新: 2025-10-19 18:59:21  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/iw8zz5hzzskiouxp>