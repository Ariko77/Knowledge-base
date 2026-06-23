# calldata

组成：

1. 函数选择器
2. data

一种数据位置标识符，主要用于**external外部参数**

它表示这些参数是：

* **只读（read-only）**\*\* 的；\*\*
* **不会被复制到内存中**\*\*（memory）；\*\*
* **直接引用外部调用数据**\*\*（即 **<code>**msg.data**</code>** 的一部分）；\*\*
* **存储在 ****调用者提供的输入数据中（call data segment）****；**
* **更省 gas（因为避免了复制）；**


> 更新: 2025-08-30 20:13:29  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ep22rf1ri8yzqro1>