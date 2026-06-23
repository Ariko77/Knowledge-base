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


> 更新: 2026-03-05 20:13:55  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/phovixo9upi39b0c>