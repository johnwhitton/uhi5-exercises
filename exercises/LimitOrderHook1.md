# Limit Orders Pt-1

## Mechanism Design

Let's think through, conceptually, how something like this can be designed.

At a very high level - a few things we will need for sure:

- Ability to place an order
- Ability to cancel an order after placing (if not filled yet)
- Ability to withdraw/redeem tokens after order is filled

All of these are things don't really have anything to do with Uniswap - and will be public functions directly on the hook that users can call.

But - assuming an order is placed, how - or rather, when - do we actually execute it? Let's think about this for a second.

Assume a pool of two tokens `A` and `B` and assume `A` is Token 0 and `B` is Token 1. Let's say current tick of the pool is `tick = 600`, i.e. A is a more expensive asset than B.

There are two types of "take profit" orders possible here:

1. Sell some amount of A when price of A goes up further
2. Sell some amount of B when the price of B goes up

For Case (1) - a price increase of `A` is represented by the tick of the pool increasing, since `A` is Token 0.

For Case (2) - a price increase of `B` is represented by the tick of the pool decreasing, since `B` is Token 1.

So we need to somehow keep track of how the tick value is changing.

When do tick values change? When a swap happens

So - roughly, the way this will work is:

1. Alice places an order on the hook to sell Token A for Token B at a certain tick greater than current tick
2. Bob comes around and does a normal swap in the hook to buy Token A and sell Token B
3. Bob's swap makes the price of Token A go up
4. `afterSwap` we check if the tick changed enough to be able to execute Alice's order
5. If yes, hook conducts another swap - fulfilling Alice's order
6. Therefore, Bob's transaction got his swap done and also Alice's order got filled

> [!NOTE] Why afterSwap?
> We use `afterSwap` for executing orders since executing the order will further move the tick in some direction. Doing it in `beforeSwap` would affect the swap Bob wanted to execute - and we don't want that. We want Bob's trade to go through normally, and execute any pending orders after his trade is done.

Also - multiple people can potentially place the exact same order. Two people can choose to sell the same token, at the same tick, in the same pool - as an order. Therefore, we need a way to keep track of how many output tokens are claimable for each user (assuming their order has been executed).

To do so, we will have our hook be a `ERC-1155` contract as well - so we can issue "claim" tokens to the users proportional to how many input tokens they provided for their order, and will use that to calculate how many output tokens they have available to claim.

## Some Assumptions

To keep things relatively simple, we'll make some assumptions and skip over certain cases (which should be resolved in a production-ready hook):

1. We are going to try and fulfill every order that exists within the range the tick moved after a swap, with zero consideration for the fact that this will increase gas costs for the original swapper - Bob. Realistically, there should be some sort of limit here probably - especially if deploying to L1 mainnet - to keep costs reasonable and not punish Bob because he happened to use a pool with this hook attached.
2. We will not consider slippage for placed orders, and allow infinite slippage. In practice, makers of the order should also be able to set some slippage limit for fulfilling their order.
3. We will not support pools with native ETH as one of the tokens in the pair. No reason not to apart from it just makes the code a bit longer, and we don't really need that just to explain how this logic works. It should be fairly simple to add support for native ETH to the hook afterwards if you'd like.

With that said, let's get started - and we'll deal with the rest as we go.

## Setting up Foundry

We know the drill from the last lesson. Let's set up Foundry, install the v4-periphery dependency, and configure it.

```bash
# Initialize a new foundry project

forge init limit-orders

# Install v4-periphery

cd limit-orders
forge install Uniswap/v4-periphery

# Place remappings

forge remappings > remappings.txt

# Remove default Counter.sol files

rm ./**/Counter_.sol
```

Now, to configure `foundry.toml` - add the following to its end:

`foundry.toml`

```toml
solc_version = '0.8.26'
evm_version = "cancun"
optimizer_runs = 800
via_ir = false
ffi = true
```

Awesome - we can get started building our hook now!

## Creating `TakeProfitsHook.sol`

Create a new file named `TakeProfitsHook.sol` under the `src/` directory - and write the following boilerplate.

`TakeProfitsHook.sol`

```Solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {BaseHook} from "v4-periphery/src/utils/BaseHook.sol";
import {ERC1155} from "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {SwapParams} from "v4-core/types/PoolOperation.sol";
import {Hooks} from "v4-core/libraries/Hooks.sol";

import {PoolId, PoolIdLibrary} from "v4-core/types/PoolId.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";

import {Currency, CurrencyLibrary} from "v4-core/types/Currency.sol";
import {StateLibrary} from "v4-core/libraries/StateLibrary.sol";

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {TickMath} from "v4-core/libraries/TickMath.sol";
import {BalanceDelta} from "v4-core/types/BalanceDelta.sol";

import {FixedPointMathLib} from "solmate/src/utils/FixedPointMathLib.sol";

contract TakeProfitsHook is BaseHook, ERC1155 {
    // StateLibrary is new here and we haven't seen that before
    // It's used to add helper functions to the PoolManager to read
    // storage values.
    // In this case, we use it for accessing `currentTick` values
    // from the pool manager
    using StateLibrary for IPoolManager;

    // PoolIdLibrary used to convert PoolKeys to IDs
    using PoolIdLibrary for PoolKey;
    // Used to represent Currency types and helper functions like `.isNative()`
    using CurrencyLibrary for Currency;
    // Used for helpful math operations like `mulDiv`
    using FixedPointMathLib for uint256;

    // Errors
    error InvalidOrder();
    error NothingToClaim();
    error NotEnoughToClaim();

    // Constructor
    constructor(IPoolManager _manager, string memory _uri) BaseHook(_manager) ERC1155(_uri) {}

    // BaseHook Functions
    function getHookPermissions() public pure override returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: false,
            afterInitialize: true,
            beforeAddLiquidity: false,
            afterAddLiquidity: false,
            beforeRemoveLiquidity: false,
            afterRemoveLiquidity: false,
            beforeSwap: false,
            afterSwap: true,
            beforeDonate: false,
            afterDonate: false,
            beforeSwapReturnDelta: false,
            afterSwapReturnDelta: false,
            afterAddLiquidityReturnDelta: false,
            afterRemoveLiquidityReturnDelta: false
        });
    }

    function _afterInitialize(address, PoolKey calldata key, uint160, int24 tick) internal override returns (bytes4) {
        // TODO
        return this.afterInitialize.selector;
    }

    function _afterSwap(address sender, PoolKey calldata key, SwapParams calldata params, BalanceDelta, bytes calldata)
        internal
        override
        returns (bytes4, int128)
    {
        // TODO
        return (this.afterSwap.selector, 0);
    }
}
```

This just sets up the basic foundation of our contract, along with the two hook functions we intend to override.

## Placing Orders

The first functionality we will build is the ability to place orders. Let's think about this conceptually:

1. Users specify which pool to place the order for, what tick to sell their tokens at, which direction the swap is happening, and how many tokens to sell
2. Since users may specify any arbitrary tick, we'll pick the closest actual usable tick based on the tick spacing of the pool - rounding down by default.
3. We save their order in storage - some sort of mapping
4. We mint them some "claim" tokens they can use to claim output tokens later on that uniquely represent their order parameters
5. We transfer the input tokens from their wallet to the hook contract

So - let's first start off by creating a little helper function for Point (2) here - getting the closest lower tick that is actually usable, given an arbitrary tick value - first.

```Solidity
function getLowerUsableTick(
    int24 tick,
    int24 tickSpacing
) private pure returns (int24) {
    // E.g. tickSpacing = 60, tick = -100
    // closest usable tick rounded-down will be -120

    // intervals = -100/60 = -1 (integer division)
    int24 intervals = tick / tickSpacing;

    // since tick < 0, we round `intervals` down to -2
    // if tick > 0, `intervals` is fine as it is
    if (tick < 0 && tick % tickSpacing != 0) intervals--; // round towards negative infinity

    // actual usable tick, then, is intervals * tickSpacing
    // i.e. -2 * 60 = -120
    return intervals * tickSpacing;

}
```

Comments in the code to help explain what is going on. Basically if `tickSpacing` is something like 60, we can only actually do swaps at ticks that are a multiple of 60 - e.g. -120, -60, 0, 60, 120, etc

If user says they want to place their order to sell at tick 100, we round down to the closest usable tick which is 60. If they want to sell at tick -100, we round down to the closest usable tick that is -120.

The next thing we need is to create a mapping to store pending orders. We'll do a nested mapping for this, which can identify their position. Add the following mapping to your contract:

```Solidity
mapping(PoolId poolId =>
    mapping(int24 tickToSellAt =>
        mapping(bool zeroForOne => uint256 inputAmount)))
            public pendingOrders;
```

So we can now do things like `pendingOrders[poolKey.toId()][tickToSellAt][zeroForOne] = inputTokensAmount`

Last helper function for now - we need to be able to represent this position as a `uint256` to use it as the Token ID for ERC-1155 claim tokens we issue to the order maker. We also need to keep track of the total supply of these claim tokens we have given out. So, let's create another mapping - and then a helper.

```Solidity
mapping(uint256 positionId => uint256 claimsSupply)
    public claimTokensSupply;
```

and the helper:

```Solidity
function getPositionId(
    PoolKey calldata key,
    int24 tick,
    bool zeroForOne
    ) public pure returns (uint256) {
    return
        uint256(keccak256(abi.encode(key.toId(), tick, zeroForOne)));
}
```

Now - we are ready to write our actual `placeOrder` function:

```Solidity
function placeOrder(
    PoolKey calldata key,
    int24 tickToSellAt,
    bool zeroForOne,
    uint256 inputAmount
) external returns (int24) {
    // Get lower actually usable tick given `tickToSellAt`
    int24 tick = getLowerUsableTick(tickToSellAt, key.tickSpacing);
    // Create a pending order
    pendingOrders[key.toId()][tick][zeroForOne] += inputAmount;

    // Mint claim tokens to user equal to their `inputAmount`
    uint256 positionId = getPositionId(key, tick, zeroForOne);
    claimTokensSupply[positionId] += inputAmount;
    _mint(msg.sender, positionId, inputAmount, "");

    // Depending on direction of swap, we select the proper input token
    // and request a transfer of those tokens to the hook contract
    address sellToken = zeroForOne
        ? Currency.unwrap(key.currency0)
        : Currency.unwrap(key.currency1);
    IERC20(sellToken).transferFrom(msg.sender, address(this), inputAmount);

    // Return the tick at which the order was actually placed
    return tick;

}
```

## Cancelling Orders

Now, let's move on to our second core functionality - being able to cancel placed orders.

With all the helpers we created for `placeOrder`, we don't need to do anything special here. This is just kind of the opposite of that function. We delete the pending order from the mapping, burn the claim tokens, reduce the claim token total supply, and send their input tokens back to them.

```Solidity
function cancelOrder(
    PoolKey calldata key,
    int24 tickToSellAt,
    bool zeroForOne,
    uint256 amountToCancel
    ) external {
    // Get lower actually usable tick for their order
    int24 tick = getLowerUsableTick(tickToSellAt, key.tickSpacing);
    uint256 positionId = getPositionId(key, tick, zeroForOne);

    // Check how many claim tokens they have for this position
    uint256 positionTokens = balanceOf(msg.sender, positionId);
    if (positionTokens < amountToCancel) revert NotEnoughToClaim();

    // Remove their `amountToCancel` worth of position from pending orders
    pendingOrders[key.toId()][tick][zeroForOne] -= amountToCancel;
    // Reduce claim token total supply and burn their share
    claimTokensSupply[positionId] -= amountToCancel;
    _burn(msg.sender, positionId, amountToCancel);

    // Send them their input token
    Currency token = zeroForOne ? key.currency0 : key.currency1;
    token.transfer(msg.sender, amountToCancel);

}
```

> [!NOTE]
> Our `cancelOrder` implementation doesn't take into account partial-cancellations. To add that, take in an additional parameter - `inputAmount` - against which `positionTokens` is compared, and only burn part of the user's claim tokens.

## Redeeming Output Tokens

We're going to jump the gun a bit here - and skip over the actual order fulfillment code for now - and handle redemption of output tokens first. Logically the actual swap code first - but that's also the longest and most involved part of this hook, so we'll leave it towards the end and spend time on it and just get done with the little functions first.

Assuming we can somehow fulfill the orders that were placed, what does it look like to redeem output tokens back out?

1. We need to store the amount of output tokens that are redeemable a specific position
2. The user has claim tokens equivalent to their input amount
3. We calculate their share of output tokens
4. Reduce that amount from the redeemable output tokens storage value
5. Burn their claim tokens
6. Transfer their output tokens to them

How do we calculate the user's share of output tokens?

Given:

- `positionTokens` = amount of claimable input tokens they have. This is equal to how many input tokens they provided.
- `totalClaimableForPosition` = amount of output tokens we have from executing this position (not necessarily just for this user, but all users who placed this order)
- `totalInputAmountForPosition` = total supply of input tokens for this position placed across limit orders we are tracking (across all users)

The user's output token amount then is a percentage of the total claimable output tokens that is proportional to their share of input tokens for this position.

User's % share of input amount = `positionTokens` / `totalInputAmountForPosition`

User's share of output tokens = `totalClaimableForPosition * (positionTokens / totalInputAmountForPosition)`

which is also equal to `(positionTokens * totalClaimableForPosition) / totalInputAmountForPosition`

With this sorted - let's first create a mapping to keep track of output token amounts:

```Solidity
mapping(uint256 positionId => uint256 outputClaimable)
public claimableOutputTokens;
```

Now, we can write our redeem function:

```Solidity
function redeem(
    PoolKey calldata key,
    int24 tickToSellAt,
    bool zeroForOne,
    uint256 inputAmountToClaimFor
    ) external {
    // Get lower actually usable tick for their order
    int24 tick = getLowerUsableTick(tickToSellAt, key.tickSpacing);
    uint256 positionId = getPositionId(key, tick, zeroForOne);

    // If no output tokens can be claimed yet i.e. order hasn't been filled
    // throw error
    if (claimableOutputTokens[positionId] == 0) revert NothingToClaim();

    // they must have claim tokens >= inputAmountToClaimFor
    uint256 positionTokens = balanceOf(msg.sender, positionId);
    if (positionTokens < inputAmountToClaimFor) revert NotEnoughToClaim();

    uint256 totalClaimableForPosition = claimableOutputTokens[positionId];
    uint256 totalInputAmountForPosition = claimTokensSupply[positionId];

    // outputAmount = (inputAmountToClaimFor * totalClaimableForPosition) / (totalInputAmountForPosition)
    uint256 outputAmount = inputAmountToClaimFor.mulDivDown(
        totalClaimableForPosition,
        totalInputAmountForPosition
    );

    // Reduce claimable output tokens amount
    // Reduce claim token total supply for position
    // Burn claim tokens
    claimableOutputTokens[positionId] -= outputAmount;
    claimTokensSupply[positionId] -= inputAmountToClaimFor;
    _burn(msg.sender, positionId, inputAmountToClaimFor);

    // Transfer output tokens
    Currency token = zeroForOne ? key.currency1 : key.currency0;
    token.transfer(msg.sender, outputAmount);

}
```

Amazing - now we can move on to the meat of this codebase!

## Executing Orders

We already know we're going to be executing our orders inside `afterSwap` - but how we do it, and the things we have to be careful about will be done later in Part 2 - since there's a few gotchas we should focus on specifically.

For Part 1 of this lesson - here - we won't actually write out the `afterSwap` hook. Instead, we'll simply create an internal function to actually execute an order (and the hook later will have logic to figure out which order and when to execute).

So, let's think about what's needed to execute an order. Assuming the order information is provided to us by a higher-level function (`afterSwap`):

1. Call `poolManager.swap` to conduct the actual swap. This will return a `BalanceDelta`
2. Settle all balances with the pool manager
3. Remove the swapped amount of input tokens from the `pendingOrders` mapping
4. Increase the amount of output tokens now claimable for this position in the `claimableOutputTokens` mapping

To keep things a bit readable - we'll make the part of "swap + settle balances" it's own function. This function will simply take in the pool key and the `SwapParams` and just call the Pool Manager and then settle balances. Let's write this first:

```Solidity
function swapAndSettleBalances(
    PoolKey calldata key,
    SwapParams memory params
    ) internal returns (BalanceDelta) {
    // Conduct the swap inside the Pool Manager
    BalanceDelta delta = poolManager.swap(key, params, "");

    // If we just did a zeroForOne swap
    // We need to send Token 0 to PM, and receive Token 1 from PM
    if (params.zeroForOne) {
        // Negative Value => Money leaving user's wallet
        // Settle with PoolManager
        if (delta.amount0() < 0) {
            _settle(key.currency0, uint128(-delta.amount0()));
        }

        // Positive Value => Money coming into user's wallet
        // Take from PM
        if (delta.amount1() > 0) {
            _take(key.currency1, uint128(delta.amount1()));
        }
    } else {
        if (delta.amount1() < 0) {
            _settle(key.currency1, uint128(-delta.amount1()));
        }

        if (delta.amount0() > 0) {
            _take(key.currency0, uint128(delta.amount0()));
        }
    }

    return delta;

}

function _settle(Currency currency, uint128 amount) internal {
    // Transfer tokens to PM and let it know
    poolManager.sync(currency);
    currency.transfer(address(poolManager), amount);
    poolManager.settle();
}

function _take(Currency currency, uint128 amount) internal {
    // Take tokens out of PM to our hook contract
    poolManager.take(currency, address(this), amount);
}
```

Now, we can build our `executeOrder` function - which given details about a specific pending order will do the swap, settle balances, and update all mappings as required:

```Solidity
function executeOrder(
    PoolKey calldata key,
    int24 tick,
    bool zeroForOne,
    uint256 inputAmount
    ) internal {
    // Do the actual swap and settle all balances
    BalanceDelta delta = swapAndSettleBalances(
        key,
        SwapParams({
        zeroForOne: zeroForOne,
        // We provide a negative value here to signify an "exact input for output" swap
        amountSpecified: -int256(inputAmount),
        // No slippage limits (maximum slippage possible)
        sqrtPriceLimitX96: zeroForOne
            ? TickMath.MIN_SQRT_PRICE + 1
            : TickMath.MAX_SQRT_PRICE - 1
        })
    );

    // `inputAmount` has been deducted from this position
    pendingOrders[key.toId()][tick][zeroForOne] -= inputAmount;
    uint256 positionId = getPositionId(key, tick, zeroForOne);
    uint256 outputAmount = zeroForOne
        ? uint256(int256(delta.amount1()))
        : uint256(int256(delta.amount0()));

    // `outputAmount` worth of tokens now can be claimed/redeemed by position holders
    claimableOutputTokens[positionId] += outputAmount;
}
```

Amazing! We're done for Part 1. In Part 2 - we will fill in the required code for `afterSwap` (and also `afterInitialize` - though that one is literally one line of code, but will make sense later).

## Testing

For now, we'll jump into setting up a basic test suite that can test our functionality for placing and cancelling orders. We cannot yet test redeem since placed orders currently have no way to actually be executed.

Create a new file named `TakeProfitsHook.t.sol` under the `test/` directory.

Now, add this boilerplate to `TakeProfitsHook.t.sol`:

`TakeProfitsHook.t.sol`

```Solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

// Foundry libraries
import {Test} from "forge-std/Test.sol";

import {Deployers} from "@uniswap/v4-core/test/utils/Deployers.sol";
import {PoolSwapTest} from "v4-core/test/PoolSwapTest.sol";
import {MockERC20} from "solmate/src/test/utils/mocks/MockERC20.sol";

import {PoolManager} from "v4-core/PoolManager.sol";
import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {SwapParams, ModifyLiquidityParams} from "v4-core/types/PoolOperation.sol";

import {PoolId, PoolIdLibrary} from "v4-core/types/PoolId.sol";
import {Currency, CurrencyLibrary} from "v4-core/types/Currency.sol";
import {StateLibrary} from "v4-core/libraries/StateLibrary.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";

import {Hooks} from "v4-core/libraries/Hooks.sol";
import {TickMath} from "v4-core/libraries/TickMath.sol";

// Our contracts
import {TakeProfitsHook} from "../src/TakeProfitsHook.sol";

contract TakeProfitsHookTest is Test, Deployers {
    // Use the libraries
    using StateLibrary for IPoolManager;
    using PoolIdLibrary for PoolKey;
    using CurrencyLibrary for Currency;

    // The two currencies (tokens) from the pool
    Currency token0;
    Currency token1;

    TakeProfitsHook hook;

    function setUp() public {
    	// TODO
    }

}
```

## setUp

In the setup function, this is generally fairly consistent across different hooks we build. This one just requires slightly more work than the Points Hook we did in one of the previous lesson:

```Solidity
function setUp() public {
    // Deploy v4 core contracts
    deployFreshManagerAndRouters();

    // Deploy two test tokens
    (token0, token1) = deployMintAndApprove2Currencies();

    // Deploy our hook
    uint160 flags = uint160(
        Hooks.AFTER_INITIALIZE_FLAG | Hooks.AFTER_SWAP_FLAG
    );
    address hookAddress = address(flags);
    deployCodeTo(
        "TakeProfitsHook.sol",
        abi.encode(manager, ""),
        hookAddress
    );
    hook = TakeProfitsHook(hookAddress);

    // Approve our hook address to spend these tokens as well
    MockERC20(Currency.unwrap(token0)).approve(
        address(hook),
        type(uint256).max
    );
    MockERC20(Currency.unwrap(token1)).approve(
        address(hook),
        type(uint256).max
    );

    // Initialize a pool with these two tokens
    (key, ) = initPool(
        token0,
        token1,
        hook,
        3000,
        SQRT_PRICE_1_1,
        ZERO_BYTES
    );

    // Add initial liquidity to the pool

    // Some liquidity from -60 to +60 tick range
    modifyLiquidityRouter.modifyLiquidity(
        key,
        ModifyLiquidityParams({
            tickLower: -60,
            tickUpper: 60,
            liquidityDelta: 10 ether,
            salt: bytes32(0)
        }),
        ZERO_BYTES
    );
    // Some liquidity from -120 to +120 tick range
    modifyLiquidityRouter.modifyLiquidity(
        key,
        ModifyLiquidityParams({
            tickLower: -120,
            tickUpper: 120,
            liquidityDelta: 10 ether,
            salt: bytes32(0)
        }),
        ZERO_BYTES
    );
    // some liquidity for full range
    modifyLiquidityRouter.modifyLiquidity(
        key,
        ModifyLiquidityParams({
            tickLower: TickMath.minUsableTick(60),
            tickUpper: TickMath.maxUsableTick(60),
            liquidityDelta: 10 ether,
            salt: bytes32(0)
        }),
        ZERO_BYTES
    );

}
```

Awesome. The comments should be self-explanatory in there.

Note that we don't really care about computing how many tokens we're adding each time we add liquidity. Not that it's unimportant, more so, it's not very relevant for the tests.

If you want to dig deeper (purely out of curiosity) for what `liquidityDelta` represents and how it correlates to exact amount of two tokens being added as liquidity, check out Haardik's blog post here - https://hackmd.io/@vFMCdNzHQNqyPq_Rej0IIw/H12JlwSYA

## test_placeOrder

Let's create our first test to test the `placeOrder` function:

```Solidity
function test_placeOrder() public {
    // Place a zeroForOne take-profit order
    // for 10e18 token0 tokens
    // at tick 100
    int24 tick = 100;
    uint256 amount = 10e18;
    bool zeroForOne = true;

    // Note the original balance of token0 we have
    uint256 originalBalance = token0.balanceOfSelf();

    // Place the order
    int24 tickLower = hook.placeOrder(key, tick, zeroForOne, amount);

    // Note the new balance of token0 we have
    uint256 newBalance = token0.balanceOfSelf();

    // Since we deployed the pool contract with tick spacing = 60
    // i.e. the tick can only be a multiple of 60
    // the tickLower should be 60 since we placed an order at tick 100
    assertEq(tickLower, 60);

    // Ensure that our balance of token0 was reduced by `amount` tokens
    assertEq(originalBalance - newBalance, amount);

    // Check the balance of ERC-1155 tokens we received
    uint256 positionId = hook.getPositionId(key, tickLower, zeroForOne);
    uint256 tokenBalance = hook.balanceOf(address(this), positionId);

    // Ensure that we were, in fact, given ERC-1155 tokens for the order
    // equal to the `amount` of token0 tokens we placed the order for
    assertTrue(positionId != 0);
    assertEq(tokenBalance, amount);

}
```

Try running `forge test` in your terminal now. Did it work? Probably not.

At this point, our tests will fail with this error:

```bash
[FAIL. Reason: ERC1155InvalidReceiver(0x34A1D3fff3958843C43aD80F30b94c510645C316)] test_placeOrder() (gas: 93881)
```

When we place an order, the hook mints ERC-1155 tokens to represent our limit order.

Also, in Foundry, our tests are run as a Solidity contract - not as function calls through an externally owned account.

As it turns out, the ERC-1155 implementation we're using from OpenZeppelin disallows transferring to smart contracts unless smart contracts explicitly say they have support to handle them. This would not be an issue if testing in Hardhat - for example - since Hardhat tests run through an EOA account - but is an issue with Foundry.

To fix this, we need to add a couple of read-only functions to our test contract based on the OpenZeppelin ERC-1155 implementation that makes it known that our contract has support for being able to receive ERC-1155 tokens.

```Solidity
function onERC1155Received(
address,
address,
uint256,
uint256,
bytes calldata
) external pure returns (bytes4) {
return this.onERC1155Received.selector;
}

function onERC1155BatchReceived(
address,
address,
uint256[] calldata,
uint256[] calldata,
bytes calldata
) external pure returns (bytes4) {
return this.onERC1155BatchReceived.selector;
}
```

With these two in place - try running `forge test` again - and the test should be passing now.

## test_cancelOrder

Let's now create another test to test `cancelOrder` functionality.

```Solidity
function test_cancelOrder() public {
    // Place an order as earlier
    int24 tick = 100;
    uint256 amount = 10e18;
    bool zeroForOne = true;

    uint256 originalBalance = token0.balanceOfSelf();
    int24 tickLower = hook.placeOrder(key, tick, zeroForOne, amount);
    uint256 newBalance = token0.balanceOfSelf();

    assertEq(tickLower, 60);
    assertEq(originalBalance - newBalance, amount);

    // Check the balance of ERC-1155 tokens we received
    uint256 positionId = hook.getPositionId(key, tickLower, zeroForOne);
    uint256 tokenBalance = hook.balanceOf(address(this), positionId);
    assertEq(tokenBalance, amount);

    // Cancel the order
    hook.cancelOrder(key, tickLower, zeroForOne, amount);

    // Check that we received our token0 tokens back, and no longer own any ERC-1155 tokens
    uint256 finalBalance = token0.balanceOfSelf();
    assertEq(finalBalance, originalBalance);

    tokenBalance = hook.balanceOf(address(this), positionId);
    assertEq(tokenBalance, 0);

}
```

Try running `forge test` again - and this should pass too! The comments explain what's going on.

Moving On
For now, we'll end this lesson here. We can't really test our `redeem` functionality just yet - not until we write code for executing orders.

In Part 2 of this lesson - we have a few things left to cover:

1. Using `afterSwap` to figure out pending orders in-range and executing them
2. Understanding the gotchas and pitfalls that can exist with an implementation of that that's not carefully thought out
3. Creating tests to test that one or more orders can be fulfilled inside `afterSwap` depending on how the tick moves
