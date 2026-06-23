# Delegatecall的存储冲突

`delegatecall` （委托调用）是一个低级别的函数，它允许我们在主合约的上下文的情况下加载和调用另一个合约的代码。这意味着被调用合约的代码被执行，但被调用合约所做的任何状态改变实际上是在主合约的存储中进行的，而不是在被调用合约的存储中。

使用规则:

<font style="background-color:#f3bb2f;">目标合约地址.delegatecall(二进制编码);</font>

二进制编码:

```solidity
abi.encodeWithSignature("函数签名", 逗号分隔的具体参数)
```

写上函数签名的例子:

```solidity
abi.encodeWithSignature("f(uint256,address)", _x, _addr)
```

## 委托合约

```solidity
contract DelegateContract {
  address public owner;//0
  uint256 public id;//1
  uint256 public updatedAt;//2
}
```

## 主合约

```solidity
contract EntryPointContract {
  address public owner = msg.sender;//0
  uint256 public id = 5;//1
  uint256 public updatedAt = block.timestamp;//2
}
```

## 整体

```solidity
contract DelegateContract {
  address public owner;//0
  uint256 public id;//1
  uint256 public updatedAt;//2

  function setValues(uint256 _newId) public {
    id = _newId;
  }
}

contract EntryPointContract {
  address public owner = msg.sender;//0
  uint256 public id = 5;//1
  uint256 public updatedAt = block.timestamp;//2
  address delegateContract;//3

  constructor(address _delegateContract) {
    delegateContract = _delegateContract;
  }

  function delegate(uint256 _newId) public returns(bool) {
    (bool success, ) =
    delegateContract.delegatecall(abi.encodeWithSignature("setValues(uint256)",
      _newId));
    return success;
  }

}
```

`EntryPointContract`有一个构造函数，接收部署的`DelegateContract`的地址来委托它的调用，以便自己的状态被`DelegateContract`修改

用5来调用delegatecall,调用了委托合约里的`setValues(uint256)`函数

```solidity
function setValues(uint256 _newId) public {
    id = _newId;
  }
```

注意这里是委托合约通过它们在存储中的声明位置,在修改主合约的存储槽

id在主合约位于1槽,在委托合约也位于1槽,当主合委托合约试图改变id时,实际上改变的是**主合约的2槽**,而不是id这个状态变量

## 变式

如果换下边这个例子,就能比较明显的看出效果:

```solidity
contract DelegateContract {
  address public owner;
  // 注意：两个变量换了位置
  uint256 public updatedAt;
  uint256 public id;

  function setValues(uint256 _newId) public {
    id = _newId;
  }

}

contract EntryPointContract {
  address public owner = msg.sender;
  uint256 public id = 5;
  uint256 public updatedAt = block.timestamp;
  address delegateContract;

  constructor(address _delegateContract) {
    delegateContract = _delegateContract;
  }

  function delegate(uint256 _newId) public returns(bool) {
    (bool success, ) =
    delegateContract.delegatecall(abi.encodeWithSignature("setValues(uint256)",
      _newId));
    return success;
  }

}
```

把主函数的id换到了2槽,而委托函数的id依旧在1槽,这时再delegatecall,看看各变量的状态

用id=15调用delegatecall，委托合约的id在2槽，这意味着他将指向2槽这个存储中的声明位置，并不管主函数的id状态变量名

主函数id依旧=5，而位于2槽的`updatedAt`变量被赋值为了15

## **总结**

拥有相同的变量类型和名称并不能确保调用合约中的这些变量会被使用。它们需要在两个合约中以相同的顺序声明


> 更新: 2025-07-11 18:46:16  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/gaezqkzuwp92wqxr>