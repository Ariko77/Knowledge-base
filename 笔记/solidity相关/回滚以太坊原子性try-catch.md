# 回滚 以太坊原子性 try-catch

# 回滚

当交易出现异常时，就会回滚

* 存在一些特殊情况，比如未经检查的低级调用，当调用失败时并不回滚，而只是返回flash

# 以太坊的原子性

要么成功，要么失败，没有半成功一说

# requestDonation函数分析

!\[f5aef263f86d2ff1c5b143c77fa43d4]\(D:\WeChat Files\wxid\_qbtpvillendh22\FileStorage\Temp\f5aef263f86d2ff1c5b143c77fa43d4.png)

目的：尝试给 `msg.sender` 捐赠一些代币或余额

* external ：外部函数，只能由合约外部调用
* returns （bool enoughBlance)：返回一个布尔值来说明余额是否充足

## try-catch

`try-catch` 是 Solidity 0.6.0+ 引入的结构，用来安全地调用**可能失败的外部合约函数**

```solidity
try wallet.donate10(msg.sender) {
    return true;
}
```

意思是，尝试调用外部合约 `wallet` 的 `donate10` 函数，给 `msg.sender` 捐10个币。如果调用成功，说明余额足够，直接返回 `true`

```solidity
} catch (bytes memory err) {
```

如果 `wallet.donate10()` 抛出了异常，`catch` 就会捕捉到错误信息（`err` 是一个 `bytes` 类型的错误数据）

```solidity
if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
    wallet.transferRemainder(msg.sender);
    return false;	
```

这是在比对错误的哈希,作用是判断错误是不是 `NotEnoughBalance()` 这个特定的错误类型。

用哈希值是因为 **Solidity 中 **`try-catch`** 无法直接判断错误名，只能靠手动比对哈希值**。

如果余额不足就执行下两行，调用`wallet.transferRemainder()` 函数，把剩下的币都转给 `msg.sender`，然后返回 `false`


> 更新: 2025-07-09 11:31:01  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ci8p8ow4mhgkyd7b>