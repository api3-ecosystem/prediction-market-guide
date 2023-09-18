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
To get started, set up an empty hardhat project and install the following dependencies:

```bash
yarn hardhat init 
```

The project structure should look like this:

```bash
/
/
/
/
```

## Coding the Prediction Market

### Setting up the Interfaces

We'll start by coding the interface for the Market Handler, Prediction Market and the MockUSDC contracts that will be responsible for handling all the bets, buying and selling tokens on both sides and concluding the prediction on the settlement date.

We are using a simple ERC20 implementation for a mockUSDC to be used on the testnet. You can use any ERC20 token you want.

Make three new contract interfaces `IERC20.sol, IMarketHandler.sol, IPredictionMarket.sol` under the `/interfaces` directory and add the following code:

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

We can now move forward and start coding the main Prediction Market contract that will be responsible for making new markets and then concluding them on the settlement date.

Make a new file called `PredictionMarket.sol` in the `/contracts` directory and add the following code:

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

    /// @notice Called by the owner on behalf of the _caller and create a market for them.
    /// @notice Step necessary to make sure all the parameters are vaild and are true with no manipulation.
    /// @param _tokenSymbol The symbol to represent the asset we are prediction upon. Eg : BTC / ETH / XRP etc.
    /// @param _isAbove True if for a prediction the price will go above a set limit and false if otherwise.
    /// @param _deadline The timestamp when the target and current price are to be checked against.
    /// @param _basePrice The minimum cost of one 'Yes' or 'No' token for the prediction market to be created.
    /// Is a multiple of 0.01 USD or 1 cent.
    function createPrediction(
        string memory _tokenSymbol,
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
            tokenSymbol: _tokenSymbol,
            targetPricePoint: _targetPricePoint,
            isAbove: _isAbove,
            proxyAddress: _proxyAddress,
            fee: TRADING_FEE,
            timestamp: block.timestamp,
            deadline: _deadline,
            marketHandler: address(predictionMH),
            predictionTokenPrice: _basePrice,
            isActive: true
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
        bool _vote
    ) external callerIsSettlement(_msgSender()) {
        require(predictions[_predictionId].deadline > block.timestamp);

        address associatedMHAddress = predictions[_predictionId].marketHandler;
        IMarketHandler mhInstance = IMarketHandler(associatedMHAddress);

        mhInstance.concludePrediction_3(_vote);

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

    receive() external payable {}

    fallback() external payable {}
}
```

This is the main contract that will be responsible for creating new markets and concluding them on the settlement date.

## Coding 