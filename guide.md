# How to make an on-chain Predictions Market from scratch

## Introduction
As we get deeper into the Web3 space, there has been an increase in demand for more dApps across all chains apart from catering to the usual DeFi use cases like lending protocols, AMMs, DEXs, etc. On-chain prediction markets have a lot of untapped potential, especially with the coming up of more Zk-based L2s, where there’s still a lot of TVL that can be secured.

A prediction market is a platform where people can buy and sell contracts that represent the outcome of a future event. Polymarket, one of the largest prediction market platforms on Polygon, enables users to make new markets and bet on the available markets and their outcomes.
In essence, the concept of any prediction market is simple:

- Users can come in and make a market for a future event with two outcomes and a settlement date.
- Other users can come in and bet on either side of the event before the settlement date.
- On the settlement date, users can redeem their bets and claim their reward if whatever event they predicted came out to be true.

This way, it provides a fair market for users to trade on future events. But when it comes to building something like this on-chain, it needs to be entirely trustless. To achieve this, we can implement all of the logic in a smart contract but things get messy when it comes to deciding upon the outcome of the event.
How would a trustless dApp know the outcome of a particular event? To solve this problem, on-chain prediction markets have been using Oracles.

## Oracles in a Prediction Market
As blockchain systems are entirely deterministic and it’s impossible to make external API calls without reaching a network consensus, oracles are essential in bringing off-chain data on-chain securely.
Read more about this here.

Polymarket uses Uma’s optimistic oracles as a resolution source for its markets. It’s basically an escalation game between contracts that initiate a request on-chain. The answer proposed by asserters can get challenged by a disputer. If a dispute is raised, the request is then sent to the DVM (Data Verification Mechanism) for voting. It can take upto 48-96 hours for the voting period to conclude and settle on the correct outcome. If the answer submitted on-chain by an asserter is not disputed, it is considered to be true. That’s how Polymarket decides on the outcome of their events.
You can read more about how Uma’s optimistic oracle works in detail

This particular implementation relies heavily on dispute resolution making it slow and inefficient for a prediction market by today’s standards.

In this guide, we’ll learn how to build an entire predictions market from scratch using API3's dAPIs. We’ll be deploying the contracts on Polygon Mumbai for this tutorial.

dAPIs are first-party data feeds that are sourced directly from first-party data providers. They utilize API3’s Airnode, which is a first-party oracle node run by the API provider to serve price data on-chain to DeFi protocols.
To read more about how dAPIs work.

The predictions market we'll be building will have the following features:
- Users can create a market for a future token price event with two outcomes and a settlement date.

- Other users can come in and bet on either side of the event before the settlement date.

- On the settlement date, users can redeem their bets and claim their reward if whatever event they predicted came out to be true.

- We'll be using API3's dAPIs to get the price of the token at the settlement date to resolve the market.

## Getting Started
To get started, clone the repo and install the dependencies.

```bash
git clone https://github.com/api3-ecosystem/prediction-market.git
cd prediction-market
```

```bash
yarn 
```

The project structure should look like this:

```bash
/backend
/frontend
/graph
/README.md
```

`/backend` contains all the contracts and scripts to deploy/interact with our contracts.

`/frontend` contains the frontend code for the prediction market.

`/graph` contains the subgraph code to index the prediction market.

## Coding the Prediction Market

### Setting up the Interfaces

We'll start by coding the interface for the Market Handler, Prediction Market and the MockUSDC contracts that will be responsible for handling all the bets, buying and selling tokens on both sides and concluding the prediction on the settlement date.

We are using a simple ERC20 implementation for a mock USDC to be used on the testnet. You can use any ERC20 token you want.

There are three contract interfaces `IERC20.sol, IMarketHandler.sol, IPredictionMarket.sol` under the `/interfaces` directory:

`IERC20.sol`
```solidity
//SPDX-License-Identifier:MIT
pragma solidity ^0.8.0;

interface IERC20 {
    function decimals() external view returns (uint256);

    function totalSupply() external view returns (uint256);

    function balanceOf(address account) external view returns (uint256);

    function transfer(
        address recipient,
        uint256 amount
    ) external returns (bool);

    function allowance(
        address owner,
        address spender
    ) external view returns (uint256);

    function approve(address spender, uint256 amount) external returns (bool);

    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 value);

    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );
}
```

`IMarketHandler.sol`
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface IMarketHandler {
    function swapTokenNoWithYes(uint256) external;

    function swapTokenYesWithNo(uint256) external;

    function buyNoToken(uint256) external;

    function buyYesToken(uint256) external;

    function sellNoToken(uint256) external;

    function sellYesToken(uint256) external;

    function concludePrediction_3(bool winner) external;
}
```

`IPredictionMarket.sol`
```solidity
// SPDX-License-Identifier: MIT
// OpenZeppelin Contracts (last updated v4.6.0) (token/ERC20/IERC20.sol)

pragma solidity ^0.8.0;

interface IPredictionMarket {
    function trackProgress(
        uint256 _id,
        address _caller,
        int256 _amountYes,
        int256 _amountNo
    ) external;
}
```

### Coding `PredictionMarket.sol`

We can now move forward and start understanding the main Prediction Market contract that will be responsible for making new markets and then concluding them on the settlement date.

Under `/backend/contracts/common` directory, you'll find `PredictionMarket.sol` contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../v1/MarketHandler.sol";
import "@openzeppelin/contracts/utils/Context.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";

/// @notice Structure to represent a given Prediction.
struct Prediction {
    string tokenSymbol; // The token symbol in question
    int224 targetPricePoint; // The target price point
    bool isAbove; // This boolean is responsible for defining if the prediction is below or above the price point
    address proxyAddress; // Address of the relevant proxy contract for each asset.
    uint256 fee; // 1 = 0.01%, 100 = 1%, Creator's cut which is further divided as a 20:80 ratio where 20% goes to the protcol and remaining is held by the prediction creator.
    uint256 timestamp; // Timestamp of the creation of prediction
    uint256 deadline; // Timestamp when the prediction is to end
    bool isActive; // Check if the prediction is open or closed
    address marketHandler; // The contract responsible for betting on the prediction.
    uint256 predictionTokenPrice; // The price of either of the token for a given market handler.
    int224 priceAtConclude; // Price returns from the dapi call.
}

/// @notice Error codes
error PM_Conclude_Failed();

/// @notice The centre point of Settlement and each new Market Handler
contract PredictionMarket is Context, Ownable {
    /// @notice Counter to track each new prediction
    using Counters for Counters.Counter;
    Counters.Counter private nextPredictionId;

    /// @notice To avoid DDOS by adding some cost to the creation. Can't be changed once defined.
    uint256 public constant PLATFORM_FEE = 50 * 10 ** 6;

    /// @notice 0.01% * 50 = 0.5%.
    uint256 public TRADING_FEE = 50;

    /// @notice Mapping to track each Prediction with a unique Id.
    mapping(uint256 => Prediction) private predictions;
    /// @notice Mapping to track each Prediction's API3 dAPI proxy address. Only set in a function available
    /// to the owner to restrict any other address from creating a pseudo prediction and manipulate it how they see fit.
    mapping(uint256 => address) private predictionIdToProxy;

    /// @notice To blacklist a certain address and disable their market creation feature.
    mapping(address => bool) private blacklisted;

    /// @notice Event to declare a prediction market is available to be traded.
    /// @param predictionId The unique identifier of the prediction.
    /// @param marketHandler The address of the MarketHandler that enables the prediction to be traded upon.
    /// @param creator The creator responsible for creating the prediction.
    /// @param timestamp The timestamp when the prediction was created to be traded upon.
    event PredictionCreated(
        uint256 indexed predictionId,
        address indexed marketHandler,
        address creator,
        uint256 timestamp,
        uint256 indexed deadline
    );

    /// @dev WILL ADD A BACKUP FORCE_CONCLUDE() TO MAKE SURE IF BECAUSE OF SOME ERROR A CERTAIN PREDICTION WASN'T ABLE
    /// @dev TO BE CONCLUDED EVEN AFTER ALL CONDITIONS PASS THE OWNER WILL STILL BE ABLE TO FORCE THE PREDICTION TO BE
    /// @dev CONCLUDED AND ALLOW THE PARTICIPANTS TO WITHDRAW THEIR REWARDS.

    /// @notice To track if for some reason a certain prediction was not able to be concluded.
    /// @param predictionId The unique identifier of the prediction.
    /// @param isAbove The target orice was supposed to be above a set limit.
    /// @param timestamp The timestamp when conclude failed.
    /// @param priceReading The current price reading provided by a dAPI.
    /// @param priceTarget The target point that was the base for a prediction.
    event ConcludeFatalError(
        uint256 indexed predictionId,
        uint256 timestamp,
        bool isAbove,
        int224 indexed priceReading,
        int224 indexed priceTarget
    );

    event HandlerProgress(
        uint256 indexed predictionId,
        address indexed marketHandler,
        address indexed trader,
        int256 amountYes,
        int256 amountNo
    );

    event PredictionConcluded(uint256 indexed predictionId, uint256 timestamp);

    /// @notice The payment token interface
    IERC20 immutable I_USDC_CONTRACT;

    /// @notice The address that starts the chain of concluding a prediction.
    address public settlementAddress;
    /// @notice The address responsible for storing the funds collected.
    address public vaultAddress;

    /// @notice Check if the address calling the function is the settlementAddress or not
    modifier callerIsSettlement(address _caller) {
        require(_caller == settlementAddress);
        _;
    }

    modifier callerIsMarketHandler(uint256 _id, address _caller) {
        address marketHandlerAddress = predictions[_id].marketHandler;
        if (marketHandlerAddress == address(0))
            revert("Invalid Prediction Id!");
        if (marketHandlerAddress != _caller)
            revert("Caller is not the market handler.");
        _;
    }

    /// @param _usdc The payment token addresTRADING_FEEs.
    constructor(address _usdc) {
        I_USDC_CONTRACT = IERC20(_usdc);

        nextPredictionId.increment();
    }

    function bytes32ToString(
        bytes32 _bytes32Data
    ) public pure returns (string memory) {
        bytes memory bytesData = new bytes(32);
        for (uint i = 0; i < 32; i++) {
            bytesData[i] = _bytes32Data[i];
        }
        return string(bytesData);
    }

    /// @notice Called by the owner on behalf of the _caller and create a market for them.
    /// @notice Step necessary to make sure all the parameters are vaild and are true with no manipulation.
    /// @param _tokenSymbol The symbol to represent the asset we are prediction upon. Eg : BTC / ETH / XRP etc.
    /// @param _isAbove True if for a prediction the price will go above a set limit and false if otherwise.
    /// @param _deadline The timestamp when the target and current price are to be checked against.
    /// @param _basePrice The minimum cost of one 'Yes' or 'No' token for the prediction market to be created.
    /// Is a multiple of 0.01 USD or 1 cent.
    function createPrediction(
        bytes32 _tokenSymbol,
        address _proxyAddress,
        bool _isAbove,
        int224 _targetPricePoint,
        uint256 _deadline,
        uint256 _basePrice
    ) external returns (uint256) {
        /// @param _caller The address that is responsible for paying the platform a set fee and create a new prediction
        /// people can bet upon.
        address _caller = _msgSender();

        require(
            I_USDC_CONTRACT.allowance(_caller, address(this)) >= PLATFORM_FEE,
            "Allowance not set!"
        );
        require(
            _proxyAddress != address(0),
            "Can't have address zero as the proxy's address."
        );
        require(_deadline > block.timestamp, "Deadline can't be in the past.");

        uint256 predictionId = nextPredictionId.current();
        Prediction memory prediction = predictions[predictionId];

        require(prediction.timestamp == 0, "Prediction already exists.");

        bool success = I_USDC_CONTRACT.transferFrom(
            _caller,
            address(this),
            PLATFORM_FEE
        );
        if (!success) revert PM_InsufficientApprovedAmount();

        PM_MarketHandler predictionMH = new PM_MarketHandler(
            predictionId,
            TRADING_FEE,
            _deadline,
            _basePrice,
            address(I_USDC_CONTRACT),
            vaultAddress
        );

        Prediction memory toAdd = Prediction({
            tokenSymbol: bytes32ToString(_tokenSymbol),
            targetPricePoint: _targetPricePoint,
            isAbove: _isAbove,
            proxyAddress: _proxyAddress,
            fee: TRADING_FEE,
            timestamp: block.timestamp,
            deadline: _deadline,
            marketHandler: address(predictionMH),
            predictionTokenPrice: _basePrice,
            isActive: true,
            priceAtConclude: -1
        });

        predictions[predictionId] = toAdd;
        predictionIdToProxy[predictionId] = _proxyAddress;

        nextPredictionId.increment();

        emit PredictionCreated(
            predictionId,
            address(predictionMH),
            _caller,
            block.timestamp,
            _deadline
        );
        return predictionId;
    }

    /// @notice Called by the Settlement contract which concludes the prediction and returns the vote i.e if the
    /// prediction was in the favour of 'Yes' or 'No'.
    /// @param _predictionId The unique identifier for each prediction created.
    /// @param _vote The final result of the prediction.
    /// vote - True : The target price was predicted to be BELOW/ABOVE a threshold AND IS BELOW/ABOVE the threshold respectively.
    /// vote - False : The target price was predicted to be BELOW/ABOVE a threshold BUT IS ABOVE/BELOW the threshold respectively.
    function concludePrediction_2(
        uint256 _predictionId,
        bool _vote,
        address _initiator,
        int224 _readPrice
    ) external callerIsSettlement(_msgSender()) {
        require(
            predictions[_predictionId].deadline <= block.timestamp,
            "Conclude_2 Failed."
        );

        address associatedMHAddress = predictions[_predictionId].marketHandler;
        IMarketHandler mhInstance = IMarketHandler(associatedMHAddress);

        mhInstance.concludePrediction_3(_vote);

        Prediction storage current = predictions[_predictionId];
        current.isActive = false;
        current.priceAtConclude = _readPrice;

        /// Rewards for concluder
        I_USDC_CONTRACT.transfer(_initiator, 40000000);
        I_USDC_CONTRACT.transfer(vaultAddress, 10000000);

        emit PredictionConcluded(_predictionId, block.timestamp);
    }

    /// @notice Setter functions ------
    function setSettlementAddress(address _settlement) external onlyOwner {
        settlementAddress = _settlement;
    }

    function setVaultAddress(address _vault) external onlyOwner {
        vaultAddress = _vault;
    }

    function setTradingFee(uint256 _newFee) external onlyOwner {
        TRADING_FEE = _newFee;
    }

    /// @notice Getter functions ------

    function getNextPredictionId() external view returns (uint256) {
        return nextPredictionId.current();
    }

    function getPrediction(
        uint256 _predictionId
    ) external view returns (Prediction memory) {
        return predictions[_predictionId];
    }

    function getPredictions(
        uint256[] memory _ids,
        uint256 _limit
    ) external view returns (Prediction[] memory) {
        Prediction[] memory toReturn;
        for (uint256 i = 0; i < _limit; i++) {
            toReturn[i] = (predictions[_ids[i]]);
        }
        return toReturn;
    }

    function getProxyForPrediction(
        uint256 _predictionId
    ) external view returns (address) {
        return predictionIdToProxy[_predictionId];
    }

    function getProxiesForPredictions(
        uint256[] memory _ids,
        uint256 _limit
    ) external view returns (address[] memory) {
        address[] memory toReturn;
        for (uint256 i = 0; i < _limit; i++) {
            toReturn[i] = predictionIdToProxy[_ids[i]];
        }
        return toReturn;
    }

    /// SPECIAL FUNCTION ====================================================
    ///
    ///
    /// @notice Function provided to act as an aggregator and help track all the things that are happening on
    /// all of its child market handlers.
    /// @dev Is important since its harder to track each market handler on 'The Graph' separately.
    function trackProgress(
        uint256 _id,
        address _caller,
        int256 _amountYes,
        int256 _amountNo
    ) external callerIsMarketHandler(_id, _msgSender()) {
        address marketHandlerAddress = predictions[_id].marketHandler;
        emit HandlerProgress(
            _id,
            marketHandlerAddress,
            _caller,
            _amountYes,
            _amountNo
        );
    }

    /// ============

    receive() external payable {}

    fallback() external payable {}
}
```

This is the main contract that will be responsible for creating new markets and concluding them on the settlement date.

We start by defining the `Prediction` struct that will be used to store all the information about a particular prediction like the token symbol, the target price point, the settlement date, fee, etc. We also define a few mappings to track the progress of each prediction and the dAPI's proxy address for each prediction. We'll learn about the dAPI's proxy address more as we move forward.

We then define a few events to track the progress of each prediction and the conclusion of each prediction.

We then define a few modifiers to restrict access to certain functions. 

The main function of this contract is the `createPrediction` function that will be used to create new markets. It takes in a few parameters like:

- The token symbol of the asset we are predicting upon.
- The address of the dAPI's proxy contract for the asset. This will be used to read the price of the asset on the settlement date.
- If the asset is predicted to be above or below a certain price point.
- The target price point for the asset.
- The deadline timestamp for the prediction.
- The base price of the prediction token. This is the minimum cost of one 'Yes' or 'No' token for the prediction market to be created. Is a multiple of 0.01 USD or 1 cent.

On each market that is created, a `PM_MarketHandler` contract is deployed that will be responsible for handling all the bets, buying and selling tokens on both sides and concluding the prediction on the settlement date. We'll learn more about the market handler contract as we move forward. 

It then generates a unique `predictionId` for each prediction and stores all the information about the prediction in the `predictions` mapping. It also stores the dAPI's proxy address for each prediction in the `predictionIdToProxy` mapping.

The other important function will be `concludePrediction_2` function that will be called by the settlement contract to conclude the prediction on the settlement date. It takes in the `predictionId` and the `vote` i.e if the prediction came out to be true or false. It then calls the `concludePrediction_3` function on the market handler contract to conclude the prediction.

There are some other getter and setter functions that we'll be using to set the settlement address, vault address, trading fee, etc.

`trackProgress` is a special function that will be called by the market handler contract to track all the things that are happening on all of its child market handlers.

The settlement contract will use API3's dAPIs to get the price of the token on the settlement date and then call the `concludePrediction_2` function on the prediction market contract to conclude the prediction.

## Coding the Settlement Contract

We can now move forward and start understanding the settlement contract that will be responsible for concluding the prediction on the settlement date.

Under `backend/contracts/common`, you'll find `Settlement.sol` contract.

```solidity
//SPDX-License-Identifier:MIT

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@api3/contracts/v0.8/interfaces/IProxy.sol";

/// @dev Current order of settling a market :
/// Settlement : concludePrediction_1 -> PredictionMarket : conludePrediction_2 -> Each Unique MarketHandler : concludePrediction_3

/// @notice We need to track certain properties of the prediction to make sure it it concluded after the deadline only.
struct Prediction {
    string tokenSymbol;
    int224 targetPricePoint;
    bool isAbove;
    address proxyAddress;
    uint256 fee;
    uint256 timestamp;
    uint256 deadline;
    bool isActive;
    address marketHandler;
}

interface IPredictionsMarket {
    function concludePrediction_2(uint256, bool) external;

    function getNextPredictionId() external view returns (uint256);

    function getPrediction(uint256) external view returns (Prediction memory);
}

error PM_InvalidPredictionId();

/// @dev The contract is inherently a data feed reader
contract PM_Settlement is Ownable {
    /// @notice The PredictionMarket contract that acts as a middle ground for Settlement and MarketHandler
    IPredictionMarket public PredictionMarket;

    /// @param _predictionMarket The PredictionMarket Contract
    constructor(address _predictionMarket) {
        PredictionMarket = IPredictionMarket(_predictionMarket);
    }

    modifier isValidPredictionId(uint256 _id) {
        uint256 currentUpper = PredictionMarket.getNextPredictionId();
        if (_id >= currentUpper) revert PM_InvalidPredictionId();
        _;
    }

    /// @dev We can add an incentive to whoever calls it get some % of overall protocol fee for a given prediction.
    /// Note that this should come out > gas fee to run the txn in the first place. Or we use CRON job, EAC,
    /// Cloud-based scheduler.
    /// @dev Personally think that the 1st and 3rd options are good candidates.
    /// @param _predictionId The unique identifier for the prediction to be concluded.
    function concludePrediction_1(
        uint256 _predictionId
    ) external isValidPredictionId(_predictionId) {
        Prediction memory associatedPrediction = PredictionMarket.getPrediction(
            _predictionId
        );
        address associatedProxyAddress = associatedPrediction.proxyAddress;

        /// API3 FTW
        (int224 value, uint256 timestamp) = IProxy(associatedProxyAddress)
            .read();

        require(
            block.timestamp > associatedPrediction.deadline &&
                timestamp > associatedPrediction.deadline,
            "Can't run evaluation! Deadline not met."
        );

        /// @dev The price was predicted to be above the target point
        if (associatedPrediction.isAbove) {
            /// @dev And IS ABOVE the target and hence True
            if (associatedPrediction.targetPricePoint > value)
                PredictionMarket.concludePrediction_2(_predictionId, true);
                /// @dev NOT ABOVE hence False
            else PredictionMarket.concludePrediction_2(_predictionId, false);
        } else {
            if (associatedPrediction.targetPricePoint < value)
                PredictionMarket.concludePrediction_2(_predictionId, true);
            else PredictionMarket.concludePrediction_2(_predictionId, false);
        }
    }
}
```

We again make use of the `Prediction` struct to store all the information about a particular prediction. We also define a few events to track the progress of each prediction and the conclusion of each prediction.

We import the `PredictionMarket` contract to interact with it and define a few modifiers to restrict access to certain functions.

`concludePrediction_1()` is the main function that can be called by anyone to conclude the prediction on the settlement date. It takes in the `predictionId` and uses dAPIs to get the price of the token on the settlement date. It then calls the `concludePrediction_2` function on the prediction market contract to conclude the prediction. It contains all the logic to decide if the prediction came out to be true or false.

To use dAPIs, we need to use the `IProxy` interface and call the `read()` function to get the price of the token on the settlement date. It takes in the proxy address of the dAPI as a parameter and returns the latest price of the token and the timestamp.

## Coding the Market Handler Contract

The `MarketHandler` contract will be responsible for handling all the bets, buying and selling tokens on both sides and rewarding the participants on the settlement date.

Each time a new market is created on the prediction market contract, a new market handler contract gets deployed.

Under backend/contracts/v1, you'll find `MarketHandler.sol` contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../interfaces/IMarketHandler.sol";
import "../interfaces/IPredictionMarket.sol";
import "../interfaces/IERC20.sol";

import "@openzeppelin/contracts/utils/Context.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";

error PM_IsClosedForTrading();
error PM_IsOpenForTrading();
error PM_InsufficientApprovedAmount();
error PM_TokenTransferFailed();
error PM_InsufficienTradeTokens();
error PM_InvalidAmountSet();
error PM_RewardsNotAvailable();
error PM_RewardAlreadyCollected();
error PM_UserDidNotWin();

/// @notice Responsible for all the trading activity for a prediction created.
contract PM_MarketHandler is Context, Ownable, IMarketHandler {
    using Counters for Counters.Counter;

    // reserveUSDC - reserveFEE = reserveYes + reserveNo
    /// @notice Total USDC collected.
    uint256 public reserveUSDC;
    /// @notice Total platform fee collected so far.
    uint256 public reserveFEE;
    /// @notice Total USDC collected against Yes tokens.
    uint256 public reserveYes;
    /// @notice Total USDC collected against No tokens.
    uint256 public reserveNo;

    /// @notice If rewards are ready to be claimed.
    bool public RewardsClaimable;

    /// @notice The context of the side that won.
    bool public winner;

    /// @notice The id alloted in the Prediction Market contract for each prediction as predictionId.
    uint256 public immutable I_SELF_ID;
    /// @notice Price of 1 token of either side. Is a multiple of 10**4.
    uint256 public immutable I_BASE_PRICE;
    /// @notice When the market will close down.
    uint256 public immutable I_DEADLINE;
    /// @notice The decimal value set in the payment token. Is required to calculate the fee during any trading activity.
    uint256 public immutable I_DECIMALS;

    /// 100% = 10**decimals(), 0.1% = 10**decimals()/1000.
    /// This is the total fee and this further divided between the creator and the platform.
    uint256 public immutable I_FEE;

    /// @notice The interface for the payment token.
    IERC20 public immutable I_USDC_CONTRACT;
    /// @notice The vault address.
    address private I_VAULT_ADDRESS;

    IPredictionMarket private I_PREDICTION_CONTRACT;

    /// @notice Variables to track the 'Yes' token and its holders
    /// @notice The current valid index where new address is to be pushed in the yesHolders array.
    Counters.Counter private yesIndex;
    ///@notice The array that holds all the address in possession of 'Yes' token.
    address[] private yesHolders;
    /// @notice Get the index of an address in the yesHolders array
    mapping(address => uint256) private yesTokenAddressToIndex;
    /// @notice Track the amount of 'Yes' tokens held by an address
    mapping(address => uint256) private YesBalances;

    /// @notice SAME AS ABOVE BUT FOR 'NO' TOKENS.
    Counters.Counter private noIndex;
    address[] private noHolders;
    mapping(address => uint256) private noTokenAddressToIndex;
    mapping(address => uint256) private NoBalances;

    /// @notice Check if a user collected their rewards to disable multiple withdrawls
    mapping(address => bool) private rewardCollected;

    // Events
    /// @notice When a trader swaps their 'Yes' for 'No' or vica versa.
    event SwapOrder(address indexed trader, int256 amountYes, int256 amountNo);
    /// @notice When a trader buys a token either 'Yes' or 'No'.
    event BuyOrder(address indexed trader, uint256 amountYes, uint256 amountNo);
    /// @notice When a trader is looking to withdraw from the prediction and collected their invested amount.
    event SellOrder(
        address indexed trader,
        uint256 amountYes,
        uint256 amountNo
    );
    /// @notice The bool value the tells the nature of the prediction result.
    event WinnerDeclared(bool winner);
    /// @notice When a trader successfully collect their rewards.
    event RewardCollected(address indexed user, uint256 amountWon);

    /// @notice Check if the collectRewards is open to be called by the winners.
    modifier isClaimable() {
        if (!RewardsClaimable) revert PM_RewardsNotAvailable();
        _;
    }

    /// @notice Check if the market is Open for trades or not.
    modifier isOpen() {
        if (block.timestamp > I_DEADLINE) revert PM_IsClosedForTrading();
        _;
    }

    /// @notice Check if the market is Closed for trades.
    modifier isClosed() {
        if (block.timestamp <= I_DEADLINE) revert PM_IsOpenForTrading();
        _;
    }

    /// @param _id The unique predictionId set in the parent Prediction Market contract.
    // _fee * 0.01% of the tokens regardless of the decimals value. Should be a natural number N.
    /// @param _fee The multiple of 0.01% to declare the final trading fee the platform collects.
    /// @param _deadline The timestamp upto which the market is open for trades.
    // 10**6 = 1 USDC, 10**4 = 0.01 USDC or 1 Cent. Therefore base price = A x 1 cent.
    /// @param _basePrice Multiple of 1 cent. Price of either of the token.
    /// @param _usdcTokenAddress The payment token address.
    /// @param _vaultAddress The vault address that will collect the reserveFEE on conclude.
    constructor(
        uint256 _id,
        uint256 _fee,
        uint256 _deadline,
        uint256 _basePrice,
        address _usdcTokenAddress,
        address _vaultAddress
    ) {
        I_PREDICTION_CONTRACT = IPredictionMarket(_msgSender());

        I_SELF_ID = _id;
        I_BASE_PRICE = _basePrice * 10 ** 4;
        I_DEADLINE = _deadline;
        I_VAULT_ADDRESS = _vaultAddress;
        IERC20 usdcContract = IERC20(_usdcTokenAddress);
        I_USDC_CONTRACT = usdcContract;
        I_DECIMALS = 10 ** usdcContract.decimals();
        I_FEE = (_fee * 10 ** usdcContract.decimals()) / 10 ** 4;

        yesIndex.increment();
        noHolders.push(address(0));
        noIndex.increment();
        yesHolders.push(address(0));
    }

    /// @notice No => Yes
    /// @notice Function To Swap 'No' tokens for 'Yes' tokens.
    /// @param _amount The amount of tokens to swap. Is a natural number N.
    function swapTokenNoWithYes(uint256 _amount) external override isOpen {
        uint256 equivalentUSDC = (_amount * I_BASE_PRICE) / I_DECIMALS;
        uint256 swapFee = getFee(equivalentUSDC);

        uint256 swapTokenDeduction = getFee(_amount);

        if (NoBalances[_msgSender()] < _amount)
            revert PM_InsufficienTradeTokens();

        NoBalances[_msgSender()] -= _amount;
        reserveNo -= equivalentUSDC;
        YesBalances[_msgSender()] += _amount - swapTokenDeduction;
        reserveYes += equivalentUSDC - swapFee;

        int256 amountYes = int256(_amount - swapTokenDeduction);
        int256 amountNo = int256(_amount);

        I_USDC_CONTRACT.transfer(I_VAULT_ADDRESS, swapFee);
        reserveFEE += swapFee;

        I_PREDICTION_CONTRACT.trackProgress(
            I_SELF_ID,
            _msgSender(),
            amountYes,
            -1 * amountNo
        );
        emit SwapOrder(_msgSender(), amountYes, -1 * amountNo);
    }

    /// @notice Yes => No
    /// @notice SAME AS ABOVE BUT TO SWAP 'Yes' for 'No' tokens.
    function swapTokenYesWithNo(uint256 _amount) external override isOpen {
        uint256 equivalentUSDC = (_amount * I_BASE_PRICE) / I_DECIMALS;
        uint256 swapFee = getFee(equivalentUSDC);

        uint256 swapTokenDeduction = getFee(_amount);

        if (YesBalances[_msgSender()] < _amount)
            revert PM_InsufficienTradeTokens();

        NoBalances[_msgSender()] += _amount - swapTokenDeduction;
        reserveNo += equivalentUSDC - swapFee;
        YesBalances[_msgSender()] -= _amount;
        reserveYes -= equivalentUSDC;

        int256 amountYes = int256(_amount);
        int256 amountNo = int256(_amount - swapTokenDeduction);

        I_USDC_CONTRACT.transfer(I_VAULT_ADDRESS, swapFee);
        reserveFEE += swapFee;

        I_PREDICTION_CONTRACT.trackProgress(
            I_SELF_ID,
            _msgSender(),
            -1 * amountYes,
            amountNo
        );
        emit SwapOrder(_msgSender(), -1 * amountYes, amountNo);
    }

    /// @notice To enable the purchase of _amount of 'No' tokens. The total fee is based on _amount * I_BASE_PRICE.
    /// @param _amount Total tokens the trader is looking to buy. Is a multiple of 10**decimals()
    function buyNoToken(uint256 _amount) external override isOpen {
        if (_amount < 1) revert();

        uint256 owedAmount = (_amount * I_BASE_PRICE) / I_DECIMALS;

        if (I_USDC_CONTRACT.allowance(_msgSender(), address(this)) < owedAmount)
            revert PM_InsufficientApprovedAmount();
        bool success = I_USDC_CONTRACT.transferFrom(
            _msgSender(),
            address(this),
            owedAmount
        );
        if (!success) revert PM_TokenTransferFailed();

        uint256 fee = getFee(owedAmount);
        I_USDC_CONTRACT.transfer(I_VAULT_ADDRESS, fee);

        reserveFEE += fee;
        reserveUSDC += owedAmount - fee;
        reserveNo += owedAmount - fee;

        uint256 finalAmount = _amount - getFee(_amount);
        NoBalances[_msgSender()] += finalAmount;

        if (noTokenAddressToIndex[_msgSender()] == 0) {
            uint256 index = noIndex.current();

            noTokenAddressToIndex[_msgSender()] = index;
            noHolders.push(_msgSender());

            noIndex.increment();
        }

        I_PREDICTION_CONTRACT.trackProgress(
            I_SELF_ID,
            _msgSender(),
            0,
            int256(finalAmount)
        );
        emit BuyOrder(_msgSender(), 0, finalAmount);
    }

    /// @notice SAME AS ABOVE BUT TO PURCHASE 'Yes' TOKENS.
    function buyYesToken(uint256 _amount) external override isOpen {
        if (_amount < 1) revert();

        uint256 owedAmount = (_amount * I_BASE_PRICE) / I_DECIMALS;

        if (I_USDC_CONTRACT.allowance(_msgSender(), address(this)) < owedAmount)
            revert PM_InsufficientApprovedAmount();
        bool success = I_USDC_CONTRACT.transferFrom(
            _msgSender(),
            address(this),
            owedAmount
        );
        if (!success) revert PM_TokenTransferFailed();

        uint256 fee = getFee(owedAmount);
        I_USDC_CONTRACT.transfer(I_VAULT_ADDRESS, fee);

        reserveFEE += fee;
        reserveUSDC += owedAmount - fee;
        reserveYes += owedAmount - fee;

        uint256 finalAmount = _amount - getFee(_amount);
        YesBalances[_msgSender()] += finalAmount;

        if (yesTokenAddressToIndex[_msgSender()] == 0) {
            uint256 index = yesIndex.current();

            yesTokenAddressToIndex[_msgSender()] = index;
            yesHolders.push(_msgSender());

            yesIndex.increment();
        }

        I_PREDICTION_CONTRACT.trackProgress(
            I_SELF_ID,
            _msgSender(),
            int256(finalAmount),
            0
        );
        emit BuyOrder(_msgSender(), finalAmount, 0);
    }

    /// @notice Function that allows a trader to dump their tokens.
    /// @param _amount The amount of 'No' Tokens the trader is willing to dump.
    function sellNoToken(uint256 _amount) external override isOpen {
        uint256 totalAmount = (_amount * I_BASE_PRICE) / I_DECIMALS;

        if (NoBalances[_msgSender()] < _amount) revert PM_InvalidAmountSet();

        uint256 fee = getFee(totalAmount);
        I_USDC_CONTRACT.transfer(I_VAULT_ADDRESS, fee);
        reserveFEE += fee;

        uint256 toSend = totalAmount - fee;
        NoBalances[_msgSender()] -= _amount;

        if (NoBalances[_msgSender()] == 0) {
            uint256 index = noTokenAddressToIndex[_msgSender()];

            noHolders[index] = address(0);
            noTokenAddressToIndex[_msgSender()] = 0;
        }

        bool success = I_USDC_CONTRACT.transfer(_msgSender(), toSend);
        if (!success) revert PM_TokenTransferFailed();

        reserveUSDC -= totalAmount;

        I_PREDICTION_CONTRACT.trackProgress(
            I_SELF_ID,
            _msgSender(),
            0,
            -1 * int256(_amount)
        );
        emit SellOrder(_msgSender(), 0, _amount);
    }

    /// @notice SAME AS ABOVE BUT FOR 'Yes' TOKENS.
    function sellYesToken(uint256 _amount) external override isOpen {
        uint256 totalAmount = (_amount * I_BASE_PRICE) / I_DECIMALS;

        if (YesBalances[_msgSender()] < _amount) revert PM_InvalidAmountSet();

        uint256 fee = getFee(totalAmount);
        I_USDC_CONTRACT.transfer(I_VAULT_ADDRESS, fee);
        reserveFEE += fee;

        uint256 toSend = totalAmount - fee;
        YesBalances[_msgSender()] -= _amount;

        if (YesBalances[_msgSender()] == 0) {
            uint256 index = yesTokenAddressToIndex[_msgSender()];

            yesHolders[index] = address(0);
            yesTokenAddressToIndex[_msgSender()] = 0;
        }

        bool success = I_USDC_CONTRACT.transfer(_msgSender(), toSend);
        if (!success) revert PM_TokenTransferFailed();

        reserveUSDC -= totalAmount;

        I_PREDICTION_CONTRACT.trackProgress(
            I_SELF_ID,
            _msgSender(),
            -1 * int256(_amount),
            0
        );
        emit SellOrder(_msgSender(), _amount, 0);
    }

    /// @notice The Prediction Market contract call this function for each individual prediction.
    /// Owner being the Prediction Market contract.
    /// @param vote The nature of the winning side.
    /// vote - True => Yes won
    /// vote - False => No won
    function concludePrediction_3(
        bool vote
    ) external override isClosed onlyOwner {
        winner = vote;
        emit WinnerDeclared(vote);

        RewardsClaimable = true;

        //// IGNORE THIS.
        // All the collected fee for the current prediction is sent back to the vault.
        // I_USDC_CONTRACT.transfer(I_VAULT_ADDRESS, reserveFEE);
    }

    /// @notice The function each winner can call to get their share of the total pool.
    /// @notice Based on  how much was the initial pool of winner token and the final pool
    /// @notice being the sum of both the winner and losing side. The final cut of the user is based on the
    /// @notice amount of tokens they held of the winning side.
    function collectRewards() external isClaimable {
        if (rewardCollected[_msgSender()]) revert PM_RewardAlreadyCollected();

        uint256 finalPool = reserveUSDC;
        uint256 initialPool;
        uint256 userTokenCount;
        uint256 userShare;

        if (winner == true) {
            if (YesBalances[_msgSender()] == 0) revert PM_UserDidNotWin();

            initialPool = reserveYes;
            userTokenCount = YesBalances[_msgSender()];
            YesBalances[_msgSender()] = 0;
        } else {
            if (NoBalances[_msgSender()] == 0) revert PM_UserDidNotWin();

            initialPool = reserveNo;
            userTokenCount = NoBalances[_msgSender()];
            NoBalances[_msgSender()] = 0;
        }

        // Calculate the final proportion of the pool they are rewarded.
        userShare = (userTokenCount * finalPool) / initialPool;

        rewardCollected[_msgSender()] = true;

        I_USDC_CONTRACT.transfer(_msgSender(), userShare);
        emit RewardCollected(_msgSender(), userShare);
    }

    /// GETTER FUNCTIONS ==========================================

    function getFee(uint256 _amount) public view returns (uint256) {
        return (_amount * I_FEE) / I_DECIMALS;
    }

    function getNoReserve() external view returns (uint256) {
        return reserveNo;
    }

    function getYesReserve() external view returns (uint256) {
        return reserveYes;
    }

    function getYesTokenCount(address _add) external view returns (uint256) {
        return YesBalances[_add];
    }

    function getNoTokenCount(address _add) external view returns (uint256) {
        return NoBalances[_add];
    }

    receive() external payable {}

    fallback() external payable {}
}
```

As this contract will get deployed each time a new prediction market is made, it inherits the parent prediction market contract's `IMarketHandler` interface.

We then define some public variables to track and check the total USDC collected, total platform fee collected, total USDC collected against YES tokens, total USDC collected against NO tokens, if rewards are ready to be claimed and the result of the side that won. We also defined events that will be emitted once a user buys a side, sells a side, swaps a side, collects their rewards, etc.

We use the I_PREDICTION_MARKET_CONTRACT interface to interact with the parent prediction market contract. We also use the `Counters` library to keep track of the number of holders of both YES and NO token for each market and some mappings to track the index of each holder and the amount of tokens they hold.

We then define the required events and modifiers to restrict access to certain functions.

The constructor takes in a few parameters like the `_id`, `_fee`, `_deadline`, `_basePrice`, `_usdcTokenAddress`, `_vaultAddress`. The `_id` is the unique identifier for each prediction created. The `_fee` is the trading fee for each market. The `_deadline` is the timestamp upto which the market is open for trades. The `_basePrice` is the price of 1 token of either side. The `_usdcTokenAddress` is the address of the payment token. The `_vaultAddress` is the address of the vault that will collect the fee when the market is concluded.

We then define the `swapTokenNoWithYes()` function that will be used to swap NO tokens for YES tokens. It takes in the `_amount` of tokens to swap and calculates the equivalent USDC value of the tokens. It then calculates the fee for the swap and the fee for the tokens. It then transfers the tokens from the user to the contract and then transfers the fee to the vault. It then updates the balances of the user and the reserves. It then emits the `SwapOrder` event.

The `swapTokenYesWithNo()` function is the same as the above function but for swapping YES tokens for NO tokens.

If a user just wants to buy YES or NO tokens for a particular market, they can use the `buyNoToken()` and `buyYesToken()` functions respectively. They take in the `_amount` of tokens to buy and calculate the equivalent USDC value of the tokens. They then check if the user has approved the contract to spend their tokens and then transfer the tokens from the user to the contract. They then calculate the fee for the tokens and transfer it to the vault. They then update the balances of the user and the reserves. They then emit the `BuyOrder` event.

If a user wants to sell their tokens, they can use the `sellNoToken()` and `sellYesToken()` functions respectively. It works the same way as the buy function but for selling the YES/NO tokens. It also emits the `SellOrder` event.

The `concludePrediction_3()` function is called by the parent prediction market contract to conclude the prediction on the settlement date. It takes in the `vote` i.e if the prediction came out to be true or false. It then updates the `winner` variable and emits the `WinnerDeclared` event. It then sets the `RewardsClaimable` variable to `true`. This is the final function that will be called to conclude the prediction.

The winners of the prediction market can call the `collectRewards()` function to collect their rewards. It calculates the final pool of the winning side and the initial pool of the winning side. It then calculates the share of the user and transfers it to them. It then emits the `RewardCollected` event.

There are some other getter functions that we'll be using to get the token reserves, token count of each user, etc.

## Coding the Vault Contract

The `PM_VAULT` contract will be responsible for collecting the fee from each market that is concluded.

```solidity
//SPDX-License-Identifier:MIT

pragma solidity ^0.8.0;

import "../interfaces/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/// @notice Vault to store reserveFee of each MarketHandler when its concluded.
contract PM_Vault is Ownable {
    /// @notice The payment token address ie USDC
    IERC20 public immutable I_USDC_CONTRACT;

    /// @param _usdcAddress Address assigned to the immutable USDC
    constructor(address _usdcAddress) {
        I_USDC_CONTRACT = IERC20(_usdcAddress);
    }

    /// @notice Get the current USDC balance of the vault
    function currentBalance() external view returns (uint256) {
        return I_USDC_CONTRACT.balanceOf(address(this));
    }

    /// @notice Transfer all the USDC present in the contract to the owner
    function sendUSDC() external {
        uint256 balance = I_USDC_CONTRACT.balanceOf(address(this));
        I_USDC_CONTRACT.transfer(owner(), balance);
    }
}
```

The constructor takes in the `_usdcAddress` and loads the USDC contract.

- `currentBalance()` returns the current USDC balance of the vault.

- `sendUSDC()` transfers all the USDC present in the contract to the owner.

We're using a `MockUSDC` contract for testing purposes. The code for the MockUSDC contract is as follows:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockUSDC is ERC20 {
    function decimals() public pure override returns (uint8) {
        return 6;
    }

    constructor() ERC20("USDC", "US Dollar Coin") {}

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }
}
```

## Deploying the Contracts

We can now move forward and deploy the contracts on a testnet. For this tutorial, we'll be using the Polygon Mumbai Testnet. You can use any testnet of your choice.

We'll be using hardhat to deploy the contracts. You can learn more about hardhat [here](https://hardhat.org/getting-started/).

The project has hardhat configured already. You can check the `hardhat.config.js` to see the configuration.

Create a `.env` file under the `/backend/` directory and add the following:

- `PK_DEPLOYER` - The private key of the account that will be used to deploy the contracts.
- `PROVIDER` - Your preferred RPC provider. For this tutorial, we'll be using the Polygon Mumbai Testnet.

Under `deploy`, you will find `0_deploy.js` file. This script will be used to deploy the contracts on the testnet.

```bash
yarn deploy
```

This command will deploy the contracts on Polygon Mumbai Testnet and print the contract addresses on the console.

```bash
Compiled 18 Solidity files successfully

Deployed MockUSDC at   : 0xEb896759724F0FC2AFd2432eEfCfDBA90413A12f
Deployed Vault at      : 0xc9D15c476017a3627254561C25930722F22d7b18
Deployed Trading at    : 0x18B9379b088013f4D465CbB46b866552a8463421
Deployed Settlement at : 0xC61c6be11E98e528683db826ea2D6C3de7f51625
```

## Interacting with the contracts

### Creating a new prediction market

After deploying the contracts, we can now interact with them. We'll be using the `PredictionMarket.sol` contract to create a new prediction market.
Head over to `scripts/createNewPrediction.js` and add the details about the market you want to create. To get the `proxyAddress` for your asset, you can use the [API3 Market](https://dapis.api3.org/). You can explore and find the proxy address for your asset.

Add in your values for `symbol`, `proxyAddress`, `isAbove`, `targetPrice`, `deadline`, `basePrice` and run the script with the following command:

```bash
yarn create-new-prediction
```

This will approve the spending limit for your Mock USDC and create a new prediction market.

### Concluding the prediction

Any user can come in and conclude a prediction if its past the deadline. To conclude a prediction, we'll be using the `Settlement.sol` contract. Head over to `scripts/concludePrediction.js` and add the `predictionId` of the prediction you want to conclude and run the script with the following command:

```bash
yarn conclude-prediction
```

This will conclude the prediction and print the result on the console.

-------- testing --------

### Buying tokens

Any user can come in and buy tokens for a particular prediction. To buy tokens, we'll be using the `MarketHandler.sol` contract. Head over to `scripts/buyYesTokens.js` or `scripts/buyNoToken.js` and add the `predictionId` of the prediction you want to buy tokens for and the `amount` of tokens you want to buy and run the script with the following command:

```bash
## For buying Yes tokens
yarn buy-yes-tokens

## For buying Yes tokens
yarn buy-no-tokens
```
