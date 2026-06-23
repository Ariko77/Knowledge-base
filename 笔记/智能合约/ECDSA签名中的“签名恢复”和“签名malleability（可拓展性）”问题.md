# ECDSA签名中的“签名恢复”和“签名 malleability（可拓展性）”问题

以太坊交易使用的是 **ECDSA**（椭圆曲线数字签名算法）。一个签名由以下三部分组成：

* `r`（32字节）
* `s`（32字节）
* `v`（1字节，用来标识哪个公钥对应这个签名）

# 签名冒充impersonation

在某些合约中，可能存在这样的验证逻辑：

```plain
address recovered = ecrecover(hash, v, r, s);
require(recovered == expectedSigner);
```

这段逻辑的意思是：**只要你能提供一个合法签名，它 recover 出来的地址是某个特定地址，那就放你过。**

# 签名可拓展性

ECDSA 签名是 **不唯一的！** 对于一个消息 `m` 和私钥 `sk`，你可以构造 **多个不同的签名**，它们都能成功通过验证。

主要原因在于：

对同一个 `(r, s)`，也可以构造 `(r, -s mod n)`，它也是合法的签名。

以太坊中，为了避免这种问题，现在规定：

* `s` 必须在 `n / 2` 以下（`n` 是椭圆曲线的阶）
* 否则签名就是非法的

但如果合约自己实现 `ecrecover` 验证，而\*\*没有验证 **<code>**s**</code>** 是否小于 \*\*<code>**n/2**</code>，那攻击者就可以“拓展”签名，**用另一个合法但不同的 **<code>**(r, s)**</code>** 对伪造签名**，从而伪装成原始签名者。

# ercrecover


> 更新: 2025-09-04 13:58:26  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/vkeohixbe8bhneev>