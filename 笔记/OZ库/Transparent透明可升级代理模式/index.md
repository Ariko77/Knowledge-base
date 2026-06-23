# Transparent透明可升级代理模式

**透明代理** =「区分 admin 和 普通用户」的调用路径：

* 管理员（admin）可以升级合约
* 普通用户（非 admin）才能调用被代理合约的业务逻辑函数

# 函数选择器冲突

## 同名函数

这里有俩合约,有同名函数`changeImplementation()`,所以**函数签名相同\*\*\*\*,且都public**

```solidity
contract ProxyUnsafe {

    function changeImplementation(
            address newImplementation
    ) public {
        // some code...
    }

    fallback(bytes calldata data) external payable (bytes memory) {
        (bool ok, bytes memory data) = getImplementation().delegatecall(data);
        require(ok, "delegatecall failed");
        return data;
    }
}
```

```solidity
contract Implementation {
    // an identical function is declared here -- they will clash
    function changeImplementation(
            address newImplementation
    ) public {

    }
    //...
}
```

这样的话,外部如果想要调用实现合约里这个changeImplementation()函数是不行的,因为会优先调用代理合约那个相同函数选择器的 public 函数,无法触发fallback,也就无法delegatecall到实现合约去调用实现合约的这个函数(delegatecall在fallback里)

* **函数选择器是由函数签名(即**<code>**functionName(paramTypes)**</code>**计算出来的4字节哈希前缀**,因此**即使函数体不同、但签名一样(包括名字和参数类型),选择器仍然一样,会发生冲突**

```solidity
bytes4(keccak256("functionName(type1,type2,...)"))
```

## 非同名函数

就算函数非同名,但仍有大约 1/42.9 亿的概率,代理合约里一个public函数的函数选择器和实现合约里那个真正想调用的函数的函数选择器相同(即使函数签名不同),也会跟上面一样的道理调用失败

例如,函数`clash550254402()`的选择器和`proxyAdmin()`是完全一样的

# immutable admin

* ERC-1967 规定：**admin 地址必须被存储在一个固定的 storage 槽位中**

**为了兼容 ERC-1967，透明可升级代理合约会把 admin 地址写入那个特定的 storage 槽位，但并不会真的用这个变量来判断权限**.在这个槽位中存在一个地址，会告诉区块浏览器：“这是一个代理合约”——这正是ERC-1967设计的目的之一.但是，如果每次调用代理时都要从 storage 中读取 admin，就会多消耗约 2100 gas, .因此，**为了节省 gas，**很多**实现合约**会用\*\* immutable 变量来保存 admin\*\*，而不是每次都从 storage读.

# 更换admin的方法

1. 指定另一个合约proxyadmin作为透明可升级代理合约的admin,写死在代理合约中
2. 透明可升级代理合约真正的admin其实是proxyadmin合约的**owner**

这样只需要修改proxyadmin合约的owner就相当于是在修改admin了

# 使代理合约无法升级(<font style="color:rgb(23, 23, 23);">non-upgradeable)</font>

如果将proxyadmin的 owner 设置为`address(0)`(零地址),或者设置为另一个**无法正确调用**<code>**upgradeToAndCall()**</code>（或无法更改 owner）的合约，那么这个 Transparent 升级代理合约将变成**不可升级的状态**。

例如：如果你把一个 `ProxyAdmin` 的 owner 设置成另一个 `ProxyAdmin`，那么除非你保留了访问链，否则可能再也无法控制它，从而导致合约**永久无法升级.**

* 一旦 owner 无法再发起升级，就等于永久锁死升级权限，这在实际部署中有时候反而是**一种安全特性**（比如项目上线后锁定逻辑）

# 合约和函数大致梳理

## proxyadmin合约

```solidity
pragma solidity ^0.8.20;

import {ITransparentUpgradeableProxy} from "./TransparentUpgradeableProxy.sol";
import {Ownable} from "../../access/Ownable.sol";

contract ProxyAdmin is Ownable {
    string public constant UPGRADE_INTERFACE_VERSION = "5.0.0";

    constructor(address initialOwner) Ownable(initialOwner) {}

    function upgradeAndCall(
        ITransparentUpgradeableProxy proxy,
        address implementation,
        bytes memory data
    ) public payable virtual onlyOwner {
        proxy.upgradeToAndCall{value: msg.value}(implementation, data);
    }
}
```

proxyadmin合约里只有一个`upgradeAndCall()`函数

* <code>**upgradeAndCall()**</code>**函数:**

ProxyAdmin合约封装了 `proxy.call(...)`的底层ABI编码与发送操作，**使外部用户不必自己encode selector和calldata**，从而安全地发起「升级 + 初始化」的操作

合约里没有 upgradeToAndCall() 函数，所有调用都是通过 fallback + selector 判断触发的

## proxy合约

```solidity
abstract contract Proxy {
    function _delegate(address implementation) internal virtual {
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            
            switch result
            case 0 {
                revert(0, returndatasize())
            }
            default {
                return(0, returndatasize())
            }
        }
    }

    function _implementation() internal view virtual returns (address);

    function _fallback() internal virtual {
        _delegate(_implementation());
    }

    fallback() external payable virtual {
        _fallback();
    }
}
```

```solidity
pragma solidity ^0.8.20;

import {Proxy} from "../Proxy.sol";
import {ERC1967Utils} from "./ERC1967Utils.sol";

contract ERC1967Proxy is Proxy {

    constructor(address implementation, bytes memory _data) payable {
        ERC1967Utils.upgradeToAndCall(implementation, _data);//设置槽位
    }

    // reads from bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1)
    function _implementation() internal view virtual override returns (address) {
        return ERC1967Utils.getImplementation();//这个函数是ERC1967里的
    }
}
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.22;
import {ERC1967Utils} from "../ERC1967/ERC1967Utils.sol";
import {ERC1967Proxy} from "../ERC1967/ERC1967Proxy.sol";
import {ProxyAdmin} from "./ProxyAdmin.sol";

contract TransparentUpgradeableProxy is ERC1967Proxy {
    address private immutable _admin;

    error ProxyDeniedAdminAccess();

    constructor(address _logic, address initialOwner, bytes memory _data) payable ERC1967Proxy(_logic, _data) {
        _admin = address(new ProxyAdmin(initialOwner));
        // Set the storage value and emit an event for ERC-1967 compatibility
        ERC1967Utils.changeAdmin(_proxyAdmin());
    }

    function _proxyAdmin() internal view virtual returns (address) {
        return _admin;
    }

    function _fallback() internal virtual override {
        if (msg.sender == _proxyAdmin()) {
            if (msg.sig != ITransparentUpgradeableProxy.upgradeToAndCall.selector) {
                revert ProxyDeniedAdminAccess();
            } else {
                _dispatchUpgradeToAndCall();
            }
        } else {
            super._fallback();
        }
    }

    function _dispatchUpgradeToAndCall() private {
        (address newImplementation, bytes memory data) = abi.decode(msg.data[4:], (address, bytes));
        ERC1967Utils.upgradeToAndCall(newImplementation, data);
    }
}
```

```solidity
** 
* @dev Performs implementation upgrade with additional setup call if data is nonempty. 
* This function is payable only if the setup call is performed, otherwise `msg.value` is rejected 
* to avoid stuck value in the contract. 
* 
* Emits an {IERC1967-Upgraded} event. 
*/

function upgradeToAndCall(address newImplementation, bytes memory data) internal {
    _setImplementation(newImplementation);
    emit IERC1967.Upgraded(newImplementation);
    if (data.length > 0) {
        Address.functionDelegateCall(newImplementation, data);
    } else {
        _checkNonPayable();
    }
}
```

* `constructor(_logic, initialOwner, _data)`:

设置实现合约，创建proxyadmin并存入storage

* `_fallback()`:

当<code><font style="color:rgb(139, 92, 246);background-color:rgb(237, 233, 254);">msg.sender</font></code><font style="color:rgb(82, 82, 82);"> == </font><code><font style="color:rgb(139, 92, 246);background-color:rgb(237, 233, 254);">_proxyAdmin()</font></code>时:

`_fallback()`函数会首先检查传入的函数选择器是不是<font style="color:rgb(139, 92, 246);background-color:rgb(237, 233, 254);">upgradeToAndCall()</font><font style="color:rgb(23, 23, 23);">函数的选择器(也就是只允许调用 </font><code><font style="color:rgb(23, 23, 23);">upgradeToAndCall()</font></code><font style="color:rgb(23, 23, 23);">函数),如果是就走</font><code><font style="color:rgb(23, 23, 23);">_dispatchUpgradeToAndCall()</font></code><font style="color:rgb(23, 23, 23);">函数来升级合约,否则就revert</font>

* `_dispatchUpgradeToAndCall()`:  <font style="color:rgb(23, 23, 23);">专门处理解码和升级逻辑 </font>

把calldata解析成`(newImpl, data)`,升级到`newImpl`, 用delegatecall调用新实现合约的初始化函数(data)

```
- decode 数据，得到 newImpl + 初始化 data
- 调用 `ERC1967Utils.upgradeToAndCall(newImpl, data)`
- 先设置实现地址为 newImpl
- 然后 delegatecall 初始化函数（比如 initialize） 
```

* \*\* **<code>**upgrade(proxy, impl)**</code>**:  \*\*

\*\* 通过 **<code>**call**</code>** 执行 **<code>**proxy.upgradeTo(newImpl)**</code>**（发起升级）  \*\*

* <code>**upgradeAndCall(proxy, impl, data)**</code>**:**

\*\* \*\*通过 ABI 编码生成调用数据：`upgradeToAndCall(newImpl, data)`，发送给 `proxy`

**关键点**：`proxy.fallback()` 中检测到是来自 `ProxyAdmin` 且是 `upgradeToAndCall` 的函数选择器，才允许进入 `_dispatchUpgradeToAndCall()`

# 基本流程

## 一、只调用

1. **普通用户** => 透明可升级代理合约(TransparentUpgradeableProxy)
2. 触发`Proxy.fallback()`
3. `TransparentUpgradeableProxy._fallback()`
4. msg.sender!=admin 进入`super._fallback()`
5. `Proxy._fallback()`, delegatecall到实现合约

```solidity
function _fallback() internal virtual override {
        if (msg.sender == _proxyAdmin()) {
            if (msg.sig != ITransparentUpgradeableProxy.upgradeToAndCall.selector) {
                revert ProxyDeniedAdminAccess();
            } else {
                _dispatchUpgradeToAndCall();
            }
        } else {
            super._fallback();
        }
    }
```

```solidity
 fallback() external payable virtual {
        _fallback();
    }
  function _fallback() internal virtual {
        _delegate(_implementation());
    }
```

## 二、升级且调用

msg.sender是`proxyadmin`,即admin

1. owner调用`proxyadmin.upgradeAndCall(proxy, impl, data)`,proxyAdmin 合约通过低级调用方式执行proxy.call(`upgradeToAndCall`.selector + encode(impl, data))发给 TransparentUpgradeableProxy合约
2. TransparentUpgradeableProxy合约没有这个函数,触发`TransparentUpgradeableProxy.fallback()`
3. `TransparentUpgradeableProxy._fallback()`
4. msg.sender == admin且函数选择器是 `upgradeToAndCall()`
5. 触发`_dispatchUpgradeToAndCall()`,解码出新代理合约地址和data,调用`ERC1967Utils.upgradeToAndCall(newImpl, data)`

```solidity
function upgradeAndCall(
        ITransparentUpgradeableProxy proxy,
        address implementation,
        bytes memory data
    ) public payable virtual onlyOwner {
        proxy.upgradeToAndCall{value: msg.value}(implementation, data);
    }
```

```solidity
 fallback() external payable virtual {
        _fallback();
    }
```

```solidity
//合约里没有upgradeToAndCall()函数,所以需要abi编码进行低级调用
function _fallback() internal virtual override {
        if (msg.sender == _proxyAdmin()) {
            if (msg.sig != ITransparentUpgradeableProxy.upgradeToAndCall.selector) {
                revert ProxyDeniedAdminAccess();
            } else {
                _dispatchUpgradeToAndCall();
            }
        } else {
            super._fallback();
        }
    }
function _dispatchUpgradeToAndCall() private {
      (address newImpl, bytes memory data) = abi.decode(msg.data[4:], (address, bytes));//解码出新代理合约地址和data
      ERC1967Utils.upgradeToAndCall(newImpl, data);//升级+调用新逻辑
    }
```

```solidity
function upgradeToAndCall(address newImplementation, bytes memory data) internal {
    _setImplementation(newImplementation);//存储槽更新为新实现合约地址
    emit IERC1967.Upgraded(newImplementation);
    if (data.length > 0) {
        Address.functionDelegateCall(newImplementation, data);
    } else {
        _checkNonPayable();
    }
}//这个函数是internal的,无法直接被继承就去用了,所以还是一开始会触发fallback
function _checkNonPayable() private {
        if (msg.value > 0) {
            revert ERC1967NonPayable();
        }
    }
```

```solidity
function functionDelegateCall(address target, bytes memory data) internal returns (bytes memory) {
        (bool success, bytes memory returndata) = target.delegatecall(data);
        return verifyCallResultFromTarget(target, success, returndata);
    }

```

只有当`data.length > 0`时,该函数才会对实现合约发起`delegatecall`

**注意，**<code>**upgradeToAndCall**</code>\*\* 并不要求升级后的实现合约必须与原来的不同 —— 它也可以“升级”到同一个实现合约\*\*

这意味着：`ProxyAdmin`合约可以通过proxy对实现合约发起任意的`delegatecall`,但从Transparent proxy的角度来看，这些调用的`msg.sender`是`ProxyAdmin`

`ProxyAdmin`能使用该合约并不构成“问题” —— 它本就有权限完全更改实现合约，且owner实际上拥有对proxy的完整管理员控制权

`ProxyAdmin`在升级操作上唯一的限制是：**不能升级到一个空合约**（即没有字节码的地址）。\
`_setImplementation()`函数会检查新的实现合约地址的字节码长度是否大于0:

```solidity
/**
 * @dev Stores a new address in the ERC-1967 implementation slot.
 */
function _setImplementation(address newImplementation) private {
    if (newImplementation.code.length == 0) {
        revert ERC1967InvalidImplementation(newImplementation);
    }
    StorageSlot.getAddressSlot(IMPLEMENTATION_SLOT).value = newImplementation;
}
```

# Summary:

1. Transparent Upgradeable Proxy 是一种设计模式，**用于避免代理合约与实现合约之间的函数选择器冲突**
2. <code>**fallback()**</code>\*\* 函数是该代理合约中唯一的公共函数\*\*
3. \*\* \*\*只有管理员才能通过 `fallback()` 函数调用升级功能；

* 非管理员的所有调用都会被作为普通调用，`delegatecall` 到实现合约；

4. 为了节省 gas，管理员地址保存在一个 `immutable` 变量中；

* 为了符合 ERC-1967 规范，合约还是会将管理员地址写入标准的 admin 槽位中，尽管运行时从不读取这个槽；

5. 由于 `immutable` 管理员地址不可变，因此**实际使用一个叫 **<code>**AdminProxy**</code>** 的智能合约作为管理员**；

* `AdminProxy` 提供一个公开函数<code>**upgradeAndCall()**</code>**，仅限其 owner 调用**；
* <code>A**dminProxy**</code>\*\* 的 owner 是可变的\*\*，谁拥有它就可以升级实现合约。


> 更新: 2025-07-17 19:54:00  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/qbbg1c3vv4lob2pf>