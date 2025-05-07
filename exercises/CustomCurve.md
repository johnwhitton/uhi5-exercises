# Custom Curve: CSMM

## Intro and Objectives

### Lesson Objectives

- Learn about Return Delta hooks
- Understand the difference between BalanceDelta and BeforeSwapDelta
- Use the beforeSwapReturnDelta hook to be able to customize swap logic for yourself
- Implement a simplified CSMM pricing curve

### Introduction

Today we're going to get started with Return Delta Hooks aka `NoOp` Hooks. For most of this lesson (and elsewhere) I will refer to them as `NoOp` hooks, so let me clarify why they're called that.

`NoOp` stands for No Operation - a Computer Science terminology that refers to the machine instruction which asks the machine to do "nothing".

In the context of Uniswap v4, `NoOp` Hooks are called such because they have the ability to ask the core `PoolManager` smart contract to do "nothing". This is a bit of a simplified statement, of course, since the `Pool Manager` is still responsible for things like calling our hook contract in the first place. Generally what we mean is being able to "skip" over a certain part of the logic within the `PoolManager`.

For today's lesson, we will build a super simple `NoOp` hook just to get started understanding the new concepts and terminologies that come with it. In following lessons, we'll start looking into more complicated stuff as well.

## Return Delta Hook Types

There are four types of Return delta hook functions:

- `beforeSwapReturnDelta`
- `afterSwapReturnDelta`
- `afterAddLiquidityReturnDelta`
- `afterRemoveLiquidityReturnDelta`

All four of these need to be used in conjunction with their "normal" counterparts. i.e. you cannot use `beforeSwapReturnDelta` if your hook also does not have the `beforeSwap` permission.

On a high level they all have different capabilities:

- `beforeSwapReturnDelta`: has the ability to partially, or completely, bypass the core swap logic of the pool manager by taking care of the swap request itself inside `beforeSwap`
- `afterSwapReturnDelta`: has the ability to extract tokens to keep for itself from the swap's output amount, ask the user to send more than the input amount of the swap with the excess going to the hook, or add additional tokens to the swap's output amount for the user
- `afterAddLiquidityReturnDelta`: has the ability to charge the user an additional amount over what they're adding as liquidity, or send them some tokens
- `afterRemoveLiquidityReturnDelta`: same as above

For today, we're going to be focused only on `beforeSwapReturnDelta` since the goal for today's lesson is to get used to a few new concepts that come with it. Also, arguably, this is one of the most powerful Return delta hook functions which will be used in most projects planning to utilize these hooks at all - so we should give it special attention.

## BeforeSwapDelta

At this point you should be familiar with `BalanceDelta` - the type returned from functions like `swap` and `modifyLiquidity`. `BalanceDelta` is of the form `(amount0, amount1)` and represents the delta of `token0` and `token1` respectively after the user performs an action. The user's responsibility is to account for this balance delta and receive the tokens it's supposed to receive, and pay the tokens it's supposed to pay, from and to the `PoolManager`.

`BeforeSwapDelta` is kind of similar. It's a distinct type that can be returned from the `beforeSwap` hook function if the `beforeSwapReturnDelta` flag has been enabled.

The first and maybe most, important distinction is in its form. Unlike `BalanceDelta` which is always of the form (amount0, amount1), `BeforeSwapDelta` is of the form `(amountSpecified, amountUnspecified)`

`amountSpecified` refers to the delta amount of the token which was "specified" by the user. `amountUnspecified` is the opposite. What does specified mean? Well, it depends on the swap parameters.

Recall that there are four different types of swap configurations that are possible:

1. Exact Input Zero for One
2. Exact Output Zero for One
3. Exact Input One for Zero
4. Exact Output One for Zero

For (1), we specify `zeroForOne = true` and `amountSpecified = a negative number` in the swap parameters. This implies that we're specifying an amount of `token0` to exactly take out of the user's wallet to use for the swap. In this case, then, the "specified" token is `token0`.

For (2), we specify `zeroForOne = true` and `amountSpecified = a positive number` in the swap parameters. This implies that we're specifying an amount of `token1` to exactly receive in the user's wallet as the output of the swap. In this case, the "specified" token is therefore `token1`.

Similarly, for (3) the specified token is `token1`, and for (4) the specified token is `token0`.

Therefore, `BeforeSwapDelta` varies from `BalanceDelta` in its format. `BalanceDelta` always has the first delta representing `token0` and the second delta representing `token1`. In `BeforeSwapDelta`, however, the first delta is for the specified token - which can be, but is not necessarily, `token0`, and the second delta is for the unspecified token - which can be, but is not necessarily, `token1`.

But why are we talking about all of this? Don't worry - we have a good reason.

Let's take a look at the `swap()` function within`PoolManager.sol`. Irrelevant code has been commented out for now.

```Solidity
/// @inheritdoc IPoolManager
function swap(PoolKey memory key, SwapParams memory params, bytes calldata hookData)
    external
    onlyWhenUnlocked
    noDelegateCall
    returns (BalanceDelta swapDelta)
    {
    // ...

    BeforeSwapDelta beforeSwapDelta;
    {
        int256 amountToSwap;
        uint24 lpFeeOverride;
        (amountToSwap, beforeSwapDelta, lpFeeOverride) = key.hooks.beforeSwap(key, params, hookData);

        // execute swap, account protocol fees, and emit swap event
        // _swap is needed to avoid stack too deep error
        swapDelta = _swap(
            pool,
            id,
            Pool.SwapParams({
                tickSpacing: key.tickSpacing,
                zeroForOne: params.zeroForOne,
                amountSpecified: amountToSwap,
                sqrtPriceLimitX96: params.sqrtPriceLimitX96,
                lpFeeOverride: lpFeeOverride
            }),
            params.zeroForOne ? key.currency0 : key.currency1 // input token
        );
    }

    // ...

}
```

Note what's happening here:

- `swap()` is called with `params`
- `params` contains the `amountSpecified` and `zeroForOne` values provided by the user
- `key.hooks.beforeSwap` is called, which returns `amountToSwap`
- Then, the internal function `_swap` is called with `amountSpecified = amountToSwap`

The important part here being the fact that the internal `_swap` function doesn't actually get called with `params.amountSpecified`. Instead, it's called with `amountToSwap` that's returned from the `key.hooks.beforeSwap` call.

Let's take a look at the `Hooks.sol` library to see what's happening inside that call. Again, some irrelevant code has been commented out :

```Solidity
/// @notice calls beforeSwap hook if permissioned and validates return value
function beforeSwap(IHooks self, PoolKey memory key, SwapParams memory params, bytes calldata hookData)
    internal
    returns (int256 amountToSwap, BeforeSwapDelta hookReturn, uint24 lpFeeOverride)
    {
    amountToSwap = params.amountSpecified;
    // ...

    if (self.hasPermission(BEFORE_SWAP_FLAG)) {
        bytes memory result = callHook(self, abi.encodeCall(IHooks.beforeSwap, (msg.sender, key, params, hookData)));

        // ...

        // skip this logic for the case where the hook return is 0
        if (self.hasPermission(BEFORE_SWAP_RETURNS_DELTA_FLAG)) {
            hookReturn = BeforeSwapDelta.wrap(result.parseReturnDelta());

            // any return in unspecified is passed to the afterSwap hook for handling
            int128 hookDeltaSpecified = hookReturn.getSpecifiedDelta();

            // Update the swap amount according to the hook's return, and check that the swap type doesnt change (exact input/output)
            if (hookDeltaSpecified != 0) {
                bool exactInput = amountToSwap < 0;
                amountToSwap += hookDeltaSpecified;
                if (exactInput ? amountToSwap > 0 : amountToSwap < 0) {
                    HookDeltaExceedsSwapAmount.selector.revertWith();
                }
            }
        }
    }

}
```

- Initially, we start off with `amountToSwap = params.amountSpecified`
- Then we call `beforeSwap` on the hook contract (assuming permission is set)
- Then we check if the hook also has permission for `beforeSwapReturnDelta`
- If yes, we extract the returned `BeforeSwapDelta` value
- If the BeforeSwapDelta value contains a "specified delta" (i.e. delta of the specified token), then we modify `amountToSwap`

This is amazing. You'll understand more how all of this comes together in a few minutes when we start working on the code. We haven't yet fully covered how it's working. The thing to note, however, is that `amountToSwap` can be modified based on the value of `BeforeSwapDelta` being returned.

This means if you want to "skip" the core swap logic in the Pool Manager, all you need to do is somehow return a `BeforeSwapDelta` that sets `amountToSwap = 0`. Alternatively, you can choose to partially skip the core swap logic by returning a `BeforeSwapDelta` such that `amountToSwap < params.amountSpecified` but not zero.

**TL;DR** - The ability to return a `BeforeSwapDelta` from the `beforeSwap` hook has implications which change the `amountToSwap` variable value, and therefore can partially or completely skip the core swap logic depending on how much `amountToSwap` is left to swap "after" the `beforeSwap` is done.

## Simplified Example

This was quite dense, so let's look at a hypothetical example. Assume a pool exists for tokens A and B where Alice wants to do an exact input swap to sell 1 Token A for B.

Recall the "regular" flow of swaps on pools with no hooks or hooks without return delta permissions only:

1. User calls `swap` on the swap router with `zeroForOne = true` and `amountSpecified = -1`
2. Swap router calls `swap` on the pool manager
3. `PoolManager` calls `beforeSwap` on the hook for whatever it needs to do (if enabled)
4. `PoolManager` calls `pools[id].swap` with `zeroForOne = true` and `amountSpecified = -1`
5. `PoolManager` gets a `BalanceDelta` of the form `(-1 A, +x B)` representing the fact that the user owes 1 Token A to the PM, and is owed some amount x of Token B from the PM
6. `PoolManager` calls `afterSwap` on hook (if enabled)
7. Swap router gets the `BalanceDelta` value and sends Token A to PM and receives Token B from PM to the user
8. Transaction complete

Now, consider a similar situation - but with `beforeSwapReturnDelta` being enabled.

1. User calls swap on swap router with `zeroForOne = true` and `amountSpecified = -1`
2. Swap router calls swap on the `PoolManager`
3. `PoolManager` calls `beforeSwap` on the hook
4. `beforeSwap` returns a `BeforeSwapDelta` of the form `(+1 A, -x B)`. The value of `x` here is not important for now, you can assume it to be zero.
5. The `BeforeSwapDelta` having +1 A implies that the hook has "consumed" the -1 A delta from the user. This sets `amountToSwap = -1` + `1` ⇒ `amountToSwap = 0`.
6. Because `amountToSwap` is zero, the core swap logic is skipped. Note that `pools[id].swap` is still called, but inside that is an if condition which checks if `amountToSwap = 0` which returns early.
7. `PoolManager` calls `afterSwap` on hook
8. Swap router gets the `BalanceDelta` value of `(-1 A, +x B)` with the amount x being dictated by our hook in this case. It settles the balances.
9. Transaction complete

In this example, by setting `BeforeSwapDelta` such that it "consumed" the user's Token A delta, we prevented the core swap logic from running. The hook could give the user some amount of Token B it calculates as well in the process - which is basically what enables the creation of custom pricing curves - but in the above example it isn't very relevant yet.

## CSMM: constant-sum market maker

Before we dig deeper, let's spend a minute quickly discussing CSMMs. We are all familiar with CPMMs already, they're AMMs which follow a constant-product pricing curve e.g. `xy = k`. A CSMM stands for constant-sum market maker, which follows a pricing curve that uses addition instead of multiplication e.g. `x + y = k`

Theoretically speaking CSMMs are great for things like stablecoins or stable assets. Imagine always being able to trade USDC vs. DAI exactly 1:1, or stETH vs ETH exactly 1:1. In practice, they're only good for hobby projects and learning because no stablecoin or stable asset is risk free, so "hardcoding" them to trade 1:1 always can be horrible (remember UST/Terra Luna?)

For this lesson though, they're great. We don't even need to go to the trouble of doing an actual `x + y = k` equation. So let's quickly break down what we're building.

We'll build a hook that enables a custom pricing curve on Uniswap v4 - but the curve is extremely simple. For any pool it's attached to, it will simply trade the assets exactly 1:1 all the time. You send 5 token 0, you get back 5 token 1. You send 5 token 1, you get back 5 token 0 - no complicated math to solve for.

## Mechanism Design

So how can we build out our custom pricing curve then using return delta hooks? Let's think through what we want the flow to be

For a swap:

1. User calls `swap` on Swap Router
2. Swap Router calls `swap` on `PoolManager`
3. `Pool Manager` calls `beforeSwap` on Hook
4. Hook should return a `BeforeSwapDelta` such that it consumes the input token, and returns an equal amount of the opposite token
5. Core swap logic gets skipped
6. `PoolManager` returns final `BalanceDelta`
7. Swap Router accounts for the balances

The key here is step (4). We've already understood how hooks can "consume" the input token - but there's still a problem here. How do we send the output token back?

Hooks cannot just magically take money out of the pool manager just because they're using a custom pricing curve. The input token in fact is going to the hook (directly or indirectly) anyway, not the Pool Manager - so the output token cannot be extracted from the Pool Manager's reserves either. Therefore, in the case of a custom pricing curve, the hook needs to manage its own liquidity that it manages based on its own pricing curve, since Uniswap's liquidity is managed only on the basis of Uniswap's pricing curve and you cannot arbitrarily withdraw funds from it because you feel like it (that'd be a huge security flaw!)

So, for adding liquidity to the pool, we need to do a couple of things:

1. Disable the default add and remove liquidity behaviour (we can do this by throwing an error from `beforeAddLiquidity` and `beforeRemoveLiquidity`)
2. Create a custom `addLiquidity` function on the hook contract that accepts tokens from users to use as liquidity in the pool

To avoid unnecessary token transfers on each swap though and also since the swap router expects to settle balances against the `PoolManager`, not the hook, we will still integrate our hook with the `PoolManager` where it will transfer the added liquidity to the `PoolManager` and only mint claim tokens in exchange. So the actual underlying tokens the user adds will be sent to the PM, and the hook will keep ERC-6909 claim tokens for itself. When a swap occurs, the hook will burn some of those ERC-6909 claim tokens and allow the Swap Router to withdraw the equivalent amount of underlying token from the `PoolManager`.

## Setting up Foundry

Let's get started with building the hook first. The full code for this project is available at https://github.com/haardikk21/csmm-noop-hook/tree/main

First, let's set up Foundry, install v4-periphery, set up remappings, and configure foundry.toml

```bash
# Initialize a new foundry project

forge init csmm-hook

# Install v4-periphery

cd csmm-hook
forge install Uniswap/v4-periphery

# Place remappings

forge remappings > remappings.txt

# Remove default Counter.sol files

rm ./**/Counter_.sol
```

Now, to configure `foundry.toml` - add the following to its end:

`foundry.toml`

```bash
solc_version = '0.8.26'
evm_version = "cancun"
optimizer_runs = 800
via_ir = false
ffi = true
```

## Creating CSMM.sol

Create a new file named `CSMM.sol` under the `src/` directory and write the following boilerplate.

`CSMM.sol`

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {SwapParams, ModifyLiquidityParams} from "v4-core/types/PoolOperation.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {PoolId} from "v4-core/types/PoolId.sol";
import {Currency} from "v4-core/types/Currency.sol";
import {CurrencySettler} from "@uniswap/v4-core/test/utils/CurrencySettler.sol";
import {Hooks} from "v4-core/libraries/Hooks.sol";
import {BeforeSwapDelta, toBeforeSwapDelta} from "v4-core/types/BeforeSwapDelta.sol";
import {BaseHook} from "v4-periphery/src/utils/BaseHook.sol";

// A CSMM is a pricing curve that follows the invariant `x + y = k`
// instead of the invariant `x * y = k`

// This is theoretically the ideal curve for a stablecoin or pegged pairs (stETH/ETH)
// In practice, we don't usually see this in prod since depegs can happen and we dont want exact equal amounts
// But is a nice little custom curve hook example

contract CSMM is BaseHook {
using CurrencySettler for Currency;

    error AddLiquidityThroughHook(); // error to throw when someone tries adding liquidity directly to the PoolManager

    event HookSwap(
        bytes32 indexed id, // v4 pool id
        address indexed sender, // router of the swap
        int128 amount0,
        int128 amount1,
        uint128 hookLPfeeAmount0,
        uint128 hookLPfeeAmount1
    );

    event HookModifyLiquidity(
        bytes32 indexed id, // v4 pool id
        address indexed sender, // router address
        int128 amount0,
        int128 amount1
    );

    constructor(IPoolManager poolManager) BaseHook(poolManager) {}

    function getHookPermissions()
        public
        pure
        override
        returns (Hooks.Permissions memory)
    {
        return
            Hooks.Permissions({
                beforeInitialize: false,
                afterInitialize: false,
                beforeAddLiquidity: true, // Don't allow adding liquidity normally
                afterAddLiquidity: false,
                beforeRemoveLiquidity: false,
                afterRemoveLiquidity: false,
                beforeSwap: true, // Override how swaps are done
                afterSwap: false,
                beforeDonate: false,
                afterDonate: false,
                beforeSwapReturnDelta: true, // Allow beforeSwap to return a custom delta
                afterSwapReturnDelta: false,
                afterAddLiquidityReturnDelta: false,
                afterRemoveLiquidityReturnDelta: false
            });
    }

    // Disable adding liquidity through the PM
    function beforeAddLiquidity(
        address,
        PoolKey calldata,
        ModifyLiquidityParams calldata,
        bytes calldata
    ) external pure override returns (bytes4) {
        revert AddLiquidityThroughHook();
    }

    // Custom add liquidity function
    function addLiquidity(PoolKey calldata key, uint256 amountEach) external {
        // TODO
    }

    function beforeSwap(
        address,
        PoolKey calldata key,
        SwapParams calldata params,
        bytes calldata
    ) external override returns (bytes4, BeforeSwapDelta, uint24) {
    	// TODO
    }

}
```

We've set up the constructor, asked for the permissions we need - `beforeSwap`, `beforeSwapReturnDelta` and `beforeAddLiquidity`. Also, we've implemented `beforeAddLiquidity` to make it revert every time someone tries to add liquidity through the `PoolManager` directly.

We've also set up stub functions for a custom `addLiquidity` function on the hook itself, and the `beforeSwap` function that we will implement shortly.

> [!NOTE]
> For the purposes of this lesson, we're not going to bother with creating receipt tokens and supporting removing liquidity from the hook. Without loss of generality it works the same way as the addition of liquidity will work, so it is left as an exercise for the reader.

## Adding Liquidity

First we'll start creating the `addLiquidity` function - since no swaps can happen until we have a way to have liquidity. Recall that we want the user to call `hook.addLiquidity()` directly - so we need to also unlock the `PoolManager` ourselves for anything we want to do with it.

First, let's create a `CallbackData` struct that can be passed to `poolManager.unlock`:

```Solidity
struct CallbackData {
    uint256 amountEach; // Amount of each token to add as liquidity
    Currency currency0;
    Currency currency1;
    address sender;
}
```

Then we can update `addLiquidity` to call `poolManager.unlock` with the callback data:

```Solidity
function addLiquidity(PoolKey calldata key, uint256 amountEach) external {
    poolManager.unlock(
        abi.encode(
        CallbackData(
            amountEach,
            key.currency0,
            key.currency1,
            msg.sender
        )
    )
    );

    emit HookModifyLiquidity(
        PoolId.unwrap(key.toId()),
        address(this),
        int128(uint128(amountEach)),
        int128(uint128(amountEach))
    );

}
```

This will trigger the Pool Manager to initiate a callback on our hook - so we need to create the callback function.

```Solidity
function unlockCallback(
    bytes calldata data
) external onlyPoolManager returns (bytes memory) {
    CallbackData memory callbackData = abi.decode(data, (CallbackData));

    // TODO: Do something with this data

    return "";

}
```

Let's revisit now what we want to do here. We want to take `amountEach` worth of tokens from the user for both currencies. That's the simple part. Later, however, when traders are conducting swaps, their balances get settled against the `PoolManager` - not the hook - so somehow we need to take the tokens from the user but store them in the `PoolManager`. But we also want the hook to "own" those tokens even if they're stored within the `PoolManager`. So how do we do this? ERC-6909 Claim Tokens!

Basically imagine the user needs to send us 500 Token A. The hook can take those 500 A and send it forward to the `PoolManager`. Note that this is just a regular token transfer, we're not triggering any actions on the PM by sending it these tokens. Once we do that, we create a debit worth 500 tokens with the `PoolManager` from the hook - so the PM owes the hook these 500 tokens back if they're not being used for anything else. Instead of taking these tokens back out though, the hook can instead mint ERC-6909 claim tokens for them representing that the hook has ownership over those 500 tokens without actually owning the underlying tokens. It's the same strategy that daytraders can use to keep their funds on the PM while only holding claim tokens.

We'll use that strategy to take money from the user in both currencies, send it all to the PM, and then mint an equivalent amount of claim tokens for both that the hook will actually keep.

```Solidity
function unlockCallback(
    bytes calldata data
) external onlyPoolManager returns (bytes memory) {
    CallbackData memory callbackData = abi.decode(data, (CallbackData));

    // Settle `amountEach` of each currency from the sender
    // i.e. Create a debit of `amountEach` of each currency with the Pool Manager
    callbackData.currency0.settle(
        poolManager,
        callbackData.sender,
        callbackData.amountEach,
        false // `burn` = `false` i.e. we're actually transferring tokens, not burning ERC-6909 Claim Tokens
    );
    callbackData.currency1.settle(
        poolManager,
        callbackData.sender,
        callbackData.amountEach,
        false
    );

    // Since we didn't go through the regular "modify liquidity" flow,
    // the PM just has a debit of `amountEach` of each currency from us
    // We can, in exchange, get back ERC-6909 claim tokens for `amountEach` of each currency
    // to create a credit of `amountEach` of each currency to us
    // that balances out the debit

    // We will store those claim tokens with the hook, so when swaps take place
    // liquidity from our CSMM can be used by minting/burning claim tokens the hook owns
    callbackData.currency0.take(
        poolManager,
        address(this),
        callbackData.amountEach,
        true // true = mint claim tokens for the hook, equivalent to money we just deposited to the PM
    );
    callbackData.currency1.take(
        poolManager,
        address(this),
        callbackData.amountEach,
        true
    );

    return "";

}
```

## Conducting Swaps

Now that liquidity can be added, we can create our `beforeSwap` to handle the swap according to our 1:1 trades pricing curve.

Here we want to do three things:

1. Get the absolute value of how many tokens the user wants to swap
2. Create a `BeforeSwapDelta` that is `(-params.amountSpecified, params.amountSpecified)` (reasoning follows shortly)
3. Account for balances, mint/burn claim tokens as needed

For explanation of steps (2) and (3), read the code comments which go into detail with specific examples:

```Solidity
function beforeSwap(
    address,
    PoolKey calldata key,
    SwapParams calldata params,
    bytes calldata
) external override returns (bytes4, BeforeSwapDelta, uint24) {
    uint256 amountInOutPositive = params.amountSpecified > 0
        ? uint256(params.amountSpecified)
        : uint256(-params.amountSpecified);

    /**
        BalanceDelta is a packed value of (currency0Amount, currency1Amount)

        BeforeSwapDelta varies such that it is not sorted by token0 and token1
        Instead, it is sorted by "specifiedCurrency" and "unspecifiedCurrency"

        Specified Currency => The currency in which the user is specifying the amount they're swapping for
        Unspecified Currency => The other currency

        For example, in an ETH/USDC pool, there are 4 possible swap cases:

        1. ETH for USDC with Exact Input for Output (amountSpecified = negative value representing ETH)
        2. ETH for USDC with Exact Output for Input (amountSpecified = positive value representing USDC)
        3. USDC for ETH with Exact Input for Output (amountSpecified = negative value representing USDC)
        4. USDC for ETH with Exact Output for Input (amountSpecified = positive value representing ETH)

        In Case (1):
            -> the user is specifying their swap amount in terms of ETH, so the specifiedCurrency is ETH
            -> the unspecifiedCurrency is USDC

        In Case (2):
            -> the user is specifying their swap amount in terms of USDC, so the specifiedCurrency is USDC
            -> the unspecifiedCurrency is ETH

        In Case (3):
            -> the user is specifying their swap amount in terms of USDC, so the specifiedCurrency is USDC
            -> the unspecifiedCurrency is ETH

        In Case (4):
            -> the user is specifying their swap amount in terms of ETH, so the specifiedCurrency is ETH
            -> the unspecifiedCurrency is USDC

        -------

        Assume zeroForOne = true (without loss of generality)
        Assume abs(amountSpecified) = 100

        For an exact input swap where amountSpecified is negative (-100)
            -> specified token = token0
            -> unspecified token = token1
            -> we set deltaSpecified = -(-100) = 100
            -> we set deltaUnspecified = -100
            -> i.e. hook is owed 100 specified token (token0) by PM (that comes from the user)
            -> and hook owes 100 unspecified token (token1) to PM (to go to the user)

        For an exact output swap where amountSpecified is positive (100)
            -> specified token = token1
            -> unspecified token = token0
            -> we set deltaSpecified = -100
            -> we set deltaUnspecified = 100
            -> i.e. hook owes 100 specified token (token1) to PM (to go to the user)
            -> and hook is owed 100 unspecified token (token0) by PM (that comes from the user)

        In either case, we can design BeforeSwapDelta as (-params.amountSpecified, params.amountSpecified)

    */

    BeforeSwapDelta beforeSwapDelta = toBeforeSwapDelta(
        int128(-params.amountSpecified), // So `specifiedAmount` = +100
        int128(params.amountSpecified) // Unspecified amount (output delta) = -100
    );

    if (params.zeroForOne) {
        // If user is selling Token 0 and buying Token 1

        // They will be sending Token 0 to the PM, creating a debit of Token 0 in the PM
        // We will take claim tokens for that Token 0 from the PM and keep it in the hook to create an equivalent credit for ourselves
        key.currency0.take(
            poolManager,
            address(this),
            amountInOutPositive,
            true
        );

        // They will be receiving Token 1 from the PM, creating a credit of Token 1 in the PM
        // We will burn claim tokens for Token 1 from the hook so PM can pay the user
        key.currency1.settle(
            poolManager,
            address(this),
            amountInOutPositive,
            true
        );

        emit HookSwap(
            PoolId.unwrap(key.toId()),
            sender,
            -int128(uint128(amountInOutPositive)),
            int128(uint128(amountInOutPositive)),
            0,
            0
        );
    } else {
        key.currency0.settle(
            poolManager,
            address(this),
            amountInOutPositive,
            true
        );
        key.currency1.take(
            poolManager,
            address(this),
            amountInOutPositive,
            true
        );

        emit HookSwap(
            PoolId.unwrap(key.toId()),
            sender,
            int128(uint128(amountInOutPositive)),
            -int128(uint128(amountInOutPositive)),
            0,
            0
        );
    }

    return (this.beforeSwap.selector, beforeSwapDelta, 0);

}
```

Hopefully the code comments make sense here.

For now, we can move on to writing our tests!

## Testing our Hook

Create a new file named `CSMM.t.sol` under the `test/` directory. We'll create a few different tests:

- Ensure that `PoolManager.modifyLiquidity` has been disabled
- Ensure liquidity can be added through `hook.addLiquidity` and that the hook mints the proper amount of claim tokens for itself
- Test exact input and exact output swaps in the `zeroForOne` direction

Start with this boilerplate that includes the setUp function:

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import {IHooks} from "v4-core/interfaces/IHooks.sol";
import {Hooks} from "v4-core/libraries/Hooks.sol";
import {TickMath} from "v4-core/libraries/TickMath.sol";
import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {SwapParams, ModifyLiquidityParams} from "v4-core/types/PoolOperation.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {BalanceDelta} from "v4-core/types/BalanceDelta.sol";
import {PoolId, PoolIdLibrary} from "v4-core/types/PoolId.sol";
import {CurrencyLibrary, Currency} from "v4-core/types/Currency.sol";
import {PoolSwapTest} from "v4-core/test/PoolSwapTest.sol";
import {Deployers} from "@uniswap/v4-core/test/utils/Deployers.sol";
import {CSMM} from "../src/CSMM.sol";
import {IERC20Minimal} from "v4-core/interfaces/external/IERC20Minimal.sol";

contract CSMMTest is Test, Deployers {
using PoolIdLibrary for PoolId;
using CurrencyLibrary for Currency;

    CSMM hook;

    function setUp() public {
        deployFreshManagerAndRouters();
        (currency0, currency1) = deployMintAndApprove2Currencies();

        address hookAddress = address(
            uint160(
                Hooks.BEFORE_ADD_LIQUIDITY_FLAG |
                    Hooks.BEFORE_SWAP_FLAG |
                    Hooks.BEFORE_SWAP_RETURNS_DELTA_FLAG
            )
        );
        deployCodeTo("CSMM.sol", abi.encode(manager), hookAddress);
        hook = CSMM(hookAddress);

        (key, ) = initPool(
            currency0,
            currency1,
            hook,
            3000,
            SQRT_PRICE_1_1,
            ZERO_BYTES
        );

        // Add some initial liquidity through the custom `addLiquidity` function
        IERC20Minimal(Currency.unwrap(key.currency0)).approve(
            hookAddress,
            1000 ether
        );
        IERC20Minimal(Currency.unwrap(key.currency1)).approve(
            hookAddress,
            1000 ether
        );

        hook.addLiquidity(key, 1000e18);
    }

}
```

### `test_cannotModifyLiquidity()`

Let's create our first test to ensure the default modify liquidity behaviour is disabled by the hook.

```Solidity
function test_cannotModifyLiquidity() public {
    vm.expectRevert();
    modifyLiquidityRouter.modifyLiquidity(
        key,
        ModifyLiquidityParams({
            tickLower: -60,
            tickUpper: 60,
            liquidityDelta: 1e18,
            salt: bytes32(0)
        }),
        ZERO_BYTES
    );
}
```

This is fairly straightforward. We use the `vm.expectRevert` cheat code to test that the `modifyLiquidity` call will revert (as it should).

### `test_claimTokenBalances()`

Now let's check if the hook actually got the claim tokens it should when we added liquidity in `setUp()`

```Solidity
function test_claimTokenBalances() public view {
// We add 1000 \* (10^18) of liquidity of each token to the CSMM pool
// The actual tokens will move into the PM
// But the hook should get equivalent amount of claim tokens for each token
uint token0ClaimID = CurrencyLibrary.toId(currency0);
uint token1ClaimID = CurrencyLibrary.toId(currency1);

    uint token0ClaimsBalance = manager.balanceOf(
        address(hook),
        token0ClaimID
    );
    uint token1ClaimsBalance = manager.balanceOf(
        address(hook),
        token1ClaimID
    );

    assertEq(token0ClaimsBalance, 1000e18);
    assertEq(token1ClaimsBalance, 1000e18);

}
```

### `test_swap_exactInput_zeroForOne()`

Let's conduct a zero for one exact input swap for 100 tokens and check that 100 token 0 got deducted from our balance and we got 100 token 1 back:

```Solidity
function test_swap_exactInput_zeroForOne() public {
    PoolSwapTest.TestSettings memory settings = PoolSwapTest.TestSettings({
        takeClaims: false,
        settleUsingBurn: false
    });

    // Swap exact input 100 Token A
    uint balanceOfTokenABefore = key.currency0.balanceOfSelf();
    uint balanceOfTokenBBefore = key.currency1.balanceOfSelf();
    swapRouter.swap(
        key,
        SwapParams({
            zeroForOne: true,
            amountSpecified: -100e18,
            sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
        }),
        settings,
        ZERO_BYTES
    );
    uint balanceOfTokenAAfter = key.currency0.balanceOfSelf();
    uint balanceOfTokenBAfter = key.currency1.balanceOfSelf();

    assertEq(balanceOfTokenBAfter - balanceOfTokenBBefore, 100e18);
    assertEq(balanceOfTokenABefore - balanceOfTokenAAfter, 100e18);

}
```

### `test_swap_exactOutput_zeroForOne()`

Lastly, we test the exact output configuration of a similar swap which should work the same way:

```Solidity
function test_swap_exactOutput_zeroForOne() public {
    PoolSwapTest.TestSettings memory settings = PoolSwapTest.TestSettings({
        takeClaims: false,
        settleUsingBurn: false
    });

    // Swap exact output 100 Token A
    uint balanceOfTokenABefore = key.currency0.balanceOfSelf();
    uint balanceOfTokenBBefore = key.currency1.balanceOfSelf();
    swapRouter.swap(
        key,
        SwapParams({
            zeroForOne: true,
            amountSpecified: 100e18,
            sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
        }),
        settings,
        ZERO_BYTES
    );
    uint balanceOfTokenAAfter = key.currency0.balanceOfSelf();
    uint balanceOfTokenBAfter = key.currency1.balanceOfSelf();

    assertEq(balanceOfTokenBAfter - balanceOfTokenBBefore, 100e18);
    assertEq(balanceOfTokenABefore - balanceOfTokenAAfter, 100e18);

}
```

Try running `forge test` now and all your tests should pass!

```bash
Ran 4 tests for test/CSMM.t.sol:CSMMTest
[PASS] test_cannotModifyLiquidity() (gas: 43101)
[PASS] test_claimTokenBalances() (gas: 20769)
[PASS] test_swap_exactInput_zeroForOne() (gas: 130752)
[PASS] test_swap_exactOutput_zeroForOne() (gas: 130676)
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 9.27ms (995.71µs CPU time)
```

## Further Improvements

The custom pricing curve we implemented is, of course, far from something you want to use in production because assuming assets will always trade 1:1 is not a great idea because of various risks that exist in the real world. The goal was to demonstrate the usage of return delta hooks - which we have done - you should use this to build better custom pricing curves than this lesson.

Our hook also doesn't have a way to remove liquidity yet. It shouldn't be too complex to create a mapping in the hook contract and store how many tokens each user added as liquidity over time to represent their share of the pool, and allow them to remove liquidity if they wish to based on that.

Our custom pricing curve also currently charges no fees. We can do this by modifying our `BeforeSwapDelta` such that it doesn't give back exactly 1:1 amounts of tokens back, but instead gives back a little less output token back. For example, 99 output tokens in exchange for 100 input tokens would be a 1% fee being charged. This fee can be distributed to the LPs of your pool.

## Conclusion

That'll be all for today. Hope you enjoyed the lesson and learnt some new stuff - I sure enjoyed doing this one. If you have any quesitons or doubts, or are planning to build this for your Capstone, feel free to ping me on the Discord server or come to my office hours and I'll try to help you out!
