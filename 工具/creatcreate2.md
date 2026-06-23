# creat create2

```solidity
function xuan(address addr) public returns (address) {
        predictaddr = address(
            uint160(
                uint(
                    keccak256(
                        abi.encodePacked(
                            bytes1(0xd6),
                            bytes1(0x94),
                            addr,
                            bytes1(0x01)//nouce=1
                        )
                    )
                )
            )
        );
        return predictaddr;
    }
```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.7.0;

import {Challenge} from "./Man.sol";
import {Attack} from "./Attack.sol";

contract Deployer {
    function computeSalt(address target) public view returns (bytes32) {
        for (uint256 i = 0; i < 1000000; i++) {
            bytes32 salt = bytes32(i);
            address predictedAddress = predictAddress(salt, target);
            if (uint256(uint160(predictedAddress)) % (16 * 16) == 239) {
                return salt; // 找salt
            }
        }
        revert("No suitable salt found within the limit");
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
```



> 更新: 2025-07-21 14:53:47  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/qn0fwcr3pf1iv2nw>