# Token 溢出漏洞

目标:手里初始有20token,用某种方法得到更多的钱,越多越好

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

//0xb5D0eF361606C246C5EF5C52319f35f226A6c27e
contract Token {
    mapping(address => uint256) balances;
    uint256 public totalSupply;

    constructor(uint256 _initialSupply) public {
        balances[msg.sender] = totalSupply = _initialSupply;
    }

    function transfer(address _to, uint256 _value) public returns (bool) {
        require(balances[msg.sender] - _value >= 0);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        return true;
    }

    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }
}
```

首先注意到**solidity版本**<code>**^0.6.0**</code>,小于^0.8.0,首先考虑**溢出漏洞**

合约里有一个`transfer(address,uint256)`函数, 那我们就可以先查一下自己余额,然后调用这个函数,把参数`_value`设的比自己余额大,就会发生下溢,钱就超级无敌多了

```solidity
cast send 0xb5D0eF361606C246C5EF5C52319f35f226A6c27e "transfer(address,uint256)" 0xb5D0eF361606C246C5EF5C52319f35f226A6c27e 21 --rpc-url $SEPOLIA_RPC --pr
ivate-key $PRIVATE_KEY
```

### 两个卡住的点

1. 纠结`_value`比`balances[msg.sender]`大,过不了require

并不是先判断`_value`和`balances[msg.sender]`谁大谁小,**require判断的是**<code>**_value**</code>**减去**<code>**balances[msg.sender]**</code>\*\*得到的数是不是大于0,\*\*也就是说先进行的是<code>**_value**</code>**减去**<code>**balances[msg.sender]**</code>**这一步，只要**<code>**_value**</code>**比**<code>**balances[msg.sender]**</code>**大,就会发生下溢得到一个超级无敌大的数，当然>0了,require肯定过呀,不要纠结在这里**(以上为小于0.8版本,在0.8版本及以上是会检查溢出的,require确实过不了)

2. 误把读槽`totalSupply`当做查询自己余额的途径了

一开始可以读槽这个看看自己有多少钱,调用`transfer()`函数打完钱以后,这个就不等于我们的余额了,所以查这个就没用了


> 更新: 2025-07-11 16:49:14  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/pxyi1ggqez59gp6s>