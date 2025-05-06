# Limit Orders Pt-2

## Introduction

Picking up from Part 1 of Limit Orders, we're now going to continue building out our hook by implementing our hook functions. Here we will see how we can keep track of tick changes and how to actually execute our orders.

So far - we have functionality for `placeOrder`, `cancelOrder`, and redeem. We also wrote an `executeOrder` function - but it isn't actually used anywhere. Let's think about what needs to happen to get it to work.

## Mechanism Design

We discussed previously that the way to execute a swap will be to check inside `afterSwap` if the tick has shifted from the last known tick, and then check if we have any pending orders within that range.

This statement gives us our first hint - we need a way to keep track of "last known" ticks for different pools. This is not information that is present directly in the `afterSwap` hook - since that will only tell us the "new"/current tick at that time. So, we must create a mapping in storage to keep track of last known ticks.

Secondly, since we are executing a swap inside `afterSwap` - this itself will trigger another execution of `afterSwap` internally. This can lead to too much recursion, re-entrancy attacks, and stack too deep errors - so we must be careful not to allow this to happen.

Thirdly - and perhaps most importantly - is recognizing that as we fulfill each order, the tick shifts even more - so we cannot simply execute all orders that existed within the original tick shift. Let's understand this one with an example - since this can be a big vulnerability.

Assume originally a pool is at tick 500, and orders have been placed by Alice at ticks 550 and 600, which will both make the tick go down if executed.

Now, Bob comes around and does a swap that increases the tick - from 500 to 650, let's say. At this point, we have two pending orders from Alice that can be fulfilled - but can we actually fulfill both?

We first come around to Alice's order at tick 550. Once we fulfill it, the tick may move to be less than 600. At this point, Alice's second order at tick 600 can no longer be fulfilled - and we need to be cognizant about this. Even though both her orders were part of the original tick shift range, as we actually executed one of her orders, the second one became no longer valid.

It is important we keep track of how the fulfillment of each individual order is updating our tick values as well - rather than just executing all orders within the original tick shift range one by one.

## afterInitialize

First, let's start off by creating a mapping to store last known tick values for different pools. Also, when a pool is initialized, its current tick is set for the first time - so we'll also instantiate that mapping inside `afterInitialize`

Add the following mapping to `TakeProfitsHook.sol`

```Solidity
// TakeProfitsHook.sol
mapping(PoolId poolId => int24 lastTick) public lastTicks;
```

and now we can write our `afterInitialize` hook function:

```Solidity
    function _afterInitialize(address, PoolKey calldata key, uint160, int24 tick) internal override returns (bytes4) {
        lastTicks[key.toId()] = tick;
        return this.afterInitialize.selector;
    }
```

That's all we need for this one.

## afterSwap

Now - let's get down to the meat of this module. Let's quickly recap the things we need to be careful about here:

1. Do not let `afterSwap` be triggered if it is being executed because of a swap our hook created while fulfilling an order (to prevent deep recursion and re-entrancy issues)
2. Identify tick shift range, find first order that can be fulfilled in that range, fill it - but then update tick shift range and search again if there are any new orders that can be fulfilled in this range or not - ignoring any orders that may have existed within the original tick shift range

This time instead of going "bottom up", we'll go "top down", since that helps explain the concept better - i.e. we'll write the helper functions we will be using later, but first write `afterSwap` itself. The following code will give you an error until we write the helper function being used here - but bear with me, and pay close attention to the code comments.

```Solidity
function _afterSwap(address sender, PoolKey calldata key, SwapParams calldata params, BalanceDelta, bytes calldata)
        internal
        override
        returns (bytes4, int128)
    {
        // `sender` is the address which initiated the swap
        // if `sender` is the hook, we don't want to go down the `afterSwap`
        // rabbit hole again
        if (sender == address(this)) return (this.afterSwap.selector, 0);

        // Should we try to find and execute orders? True initially
        bool tryMore = true;
        int24 currentTick;

        while (tryMore) {
            // Try executing pending orders for this pool

            // `tryMore` is true if we successfully found and executed an order
            // which shifted the tick value
            // and therefore we need to look again if there are any pending orders
            // within the new tick range

            // `tickAfterExecutingOrder` is the tick value of the pool
            // after executing an order
            // if no order was executed, `tickAfterExecutingOrder` will be
            // the same as current tick, and `tryMore` will be false
            (tryMore, currentTick) = tryExecutingOrders(key, !params.zeroForOne);
        }

        // New last known tick for this pool is the tick value
        // after our orders are executed
        lastTicks[key.toId()] = currentTick;
        return (this.afterSwap.selector, 0);
    }
```

Let's dissect this bit of code.

First, note that the `sender` address refers to the address that initiated the swap. When our hook fulfills an order and does a swap, the sender address is our hook contract itself. So, to prevent getting stuck in a recursive loop, we don't proceed down the path of `afterSwap` if the sender address is the hook contract itself.

Then, assuming `sender` is not the hook itself, we will set up a `while` loop which keeps trying to fulfill orders within the tick shift range.

Remember from Part 1 that we are not considering implications of potentially having hundreds or thousands of orders to execute here - causing out of gas issues - we will not have any sort of limits. In a production implementation, you might want to have some reasonable limit here on maximum number of orders you try to fulfill inside `afterSwap`.

What we do need to take care of, however, is being cognizant about the fact that each order we fulfill will update the current tick value further - making previously eligible orders no longer eligible to be executed since tick may have become too low/too high at this point.

So, inside the `while` loop - we use a helper function (not yet written) called `tryExecutingOrders`. This function will return us two values - `tryMore` and `tickAfterExecutingOrder`.

If `tryExecutingOrders` is able to find an order to execute, it will execute that order, and return `tryMore = true` along with the updated current tick value. Then, the `while` loop will iterate again, and `tryExecutingOrders` will try and find an order to execute within the new tick range.

Once `tryExecutingOrders` has no more pending orders left within the tick shift range, it will return `tryMore = false`, and we can exit our loop and exit out of `afterSwap`.

Let's start writing `tryExecutingOrders` - and that will further explain it to us.

## tryExecutingOrders

The barebones structure of this function looks something like this:

```Solidity
function tryExecutingOrders(PoolKey calldata key, bool executeZeroForOne)
        internal
        returns (bool tryMore, int24 newTick)
    {
        (, int24 currentTick,,) = poolManager.getSlot0(key.toId());
        int24 lastTick = lastTicks[key.toId()];

        // Given `currentTick` and `lastTick`, 2 cases are possible:

        // Case (1) - Tick has increased, i.e. `currentTick > lastTick`
        // or, Case (2) - Tick has decreased, i.e. `currentTick < lastTick`

        // If tick increases => Token 0 price has increased
        // => We should check if we have orders looking to sell Token 0
        // i.e. orders with zeroForOne = true

        // ------------
        // Case (1)
        // ------------

        // Tick has increased i.e. people bought Token 0 by selling Token 1
        // i.e. Token 0 price has increased
        // e.g. in an ETH/USDC pool, people are buying ETH for USDC causing ETH price to increase
        // We should check if we have any orders looking to sell Token 0
        // at ticks `lastTick` to `currentTick`
        // i.e. check if we have any orders to sell ETH at the new price that ETH is at now because of the increase

        // ------------
        // Case (2)
        // ------------
        // Tick has gone down i.e. people bought Token 1 by selling Token 0
        // i.e. Token 1 price has increased
        // e.g. in an ETH/USDC pool, people are selling ETH for USDC causing ETH price to decrease (and USDC to increase)
        // We should check if we have any orders looking to sell Token 1
        // at ticks `currentTick` to `lastTick`
        // i.e. check if we have any orders to buy ETH at the new price that ETH is at now because of the decrease
        else {
            for (int24 tick = lastTick; tick > currentTick; tick -= key.tickSpacing) {
                uint256 inputAmount = pendingOrders[key.toId()][tick][executeZeroForOne];
                if (inputAmount > 0) {
                    executeOrder(key, tick, executeZeroForOne, inputAmount);
                    return (true, currentTick);
                }
            }
        }

        return (false, currentTick);
    }
```

Within this function, once we have the values of `currentTick` and `lastTick`, we need to handle the cases of whether the tick has increased or decreased since the last known tick. This will tell us which direction of orders we should be looking to fulfill.

Let's start by handling Case (1) first - where `currentTick` > `lastTick`

```Solidity
    // ------------
    // Case (1)
    // ------------

    // Tick has increased i.e. people bought Token 0 by selling Token 1
    // i.e. Token 0 price has increased
    // e.g. in an ETH/USDC pool, people are buying ETH for USDC causing ETH price to increase
    // We should check if we have any orders looking to sell Token 0
    // at ticks `lastTick` to `currentTick`
    // i.e. check if we have any orders to sell ETH at the new price that ETH is at now because of the increase
   if (currentTick > lastTick) {
            // Loop over all ticks from `lastTick` to `currentTick`
            // and execute orders that are looking to sell Token 0
            for (int24 tick = lastTick; tick < currentTick; tick += key.tickSpacing) {
                uint256 inputAmount = pendingOrders[key.toId()][tick][executeZeroForOne];
                if (inputAmount > 0) {
                    // An order with these parameters can be placed by one or more users
                    // We execute the full order as a single swap
                    // Regardless of how many unique users placed the same order
                    executeOrder(key, tick, executeZeroForOne, inputAmount);

                    // Return true because we may have more orders to execute
                    // from lastTick to new current tick
                    // But we need to iterate again from scratch since our sale of ETH shifted the tick down
                    return (true, currentTick);
                }
            }
        }
```

Similarly, let's do the opposite direction as well - where `currentTick < lastTick`

```Solidity

        // ------------
        // Case (2)
        // ------------
        // Tick has gone down i.e. people bought Token 1 by selling Token 0
        // i.e. Token 1 price has increased
        // e.g. in an ETH/USDC pool, people are selling ETH for USDC causing ETH price to decrease (and USDC to increase)
        // We should check if we have any orders looking to sell Token 1
        // at ticks `currentTick` to `lastTick`
        // i.e. check if we have any orders to buy ETH at the new price that ETH is at now because of the decrease
        else {
            for (int24 tick = lastTick; tick > currentTick; tick -= key.tickSpacing) {
                uint256 inputAmount = pendingOrders[key.toId()][tick][executeZeroForOne];
                if (inputAmount > 0) {
                    executeOrder(key, tick, executeZeroForOne, inputAmount);
                    return (true, currentTick);
                }
            }
        }
```

Therefore, our full function now looks like:

```Solidity

    // Internal Functions
    function tryExecutingOrders(PoolKey calldata key, bool executeZeroForOne)
        internal
        returns (bool tryMore, int24 newTick)
    {
        (, int24 currentTick,,) = poolManager.getSlot0(key.toId());
        int24 lastTick = lastTicks[key.toId()];

        // Given `currentTick` and `lastTick`, 2 cases are possible:

        // Case (1) - Tick has increased, i.e. `currentTick > lastTick`
        // or, Case (2) - Tick has decreased, i.e. `currentTick < lastTick`

        // If tick increases => Token 0 price has increased
        // => We should check if we have orders looking to sell Token 0
        // i.e. orders with zeroForOne = true

        // ------------
        // Case (1)
        // ------------

        // Tick has increased i.e. people bought Token 0 by selling Token 1
        // i.e. Token 0 price has increased
        // e.g. in an ETH/USDC pool, people are buying ETH for USDC causing ETH price to increase
        // We should check if we have any orders looking to sell Token 0
        // at ticks `lastTick` to `currentTick`
        // i.e. check if we have any orders to sell ETH at the new price that ETH is at now because of the increase
        if (currentTick > lastTick) {
            // Loop over all ticks from `lastTick` to `currentTick`
            // and execute orders that are looking to sell Token 0
            for (int24 tick = lastTick; tick < currentTick; tick += key.tickSpacing) {
                uint256 inputAmount = pendingOrders[key.toId()][tick][executeZeroForOne];
                if (inputAmount > 0) {
                    // An order with these parameters can be placed by one or more users
                    // We execute the full order as a single swap
                    // Regardless of how many unique users placed the same order
                    executeOrder(key, tick, executeZeroForOne, inputAmount);

                    // Return true because we may have more orders to execute
                    // from lastTick to new current tick
                    // But we need to iterate again from scratch since our sale of ETH shifted the tick down
                    return (true, currentTick);
                }
            }
        }
        // ------------
        // Case (2)
        // ------------
        // Tick has gone down i.e. people bought Token 1 by selling Token 0
        // i.e. Token 1 price has increased
        // e.g. in an ETH/USDC pool, people are selling ETH for USDC causing ETH price to decrease (and USDC to increase)
        // We should check if we have any orders looking to sell Token 1
        // at ticks `currentTick` to `lastTick`
        // i.e. check if we have any orders to buy ETH at the new price that ETH is at now because of the decrease
        else {
            for (int24 tick = lastTick; tick > currentTick; tick -= key.tickSpacing) {
                uint256 inputAmount = pendingOrders[key.toId()][tick][executeZeroForOne];
                if (inputAmount > 0) {
                    executeOrder(key, tick, executeZeroForOne, inputAmount);
                    return (true, currentTick);
                }
            }
        }

        return (false, currentTick);
    }
```

Let's head on over to start testing our code further.

## Testing

1. We already set up some basic tests in Part 1. Today, we'll add a few more tests there. Specifically, we'll add tests for:
2. Place a zeroForOne i.e. sell Token 0 order, and make sure it's executed if tick goes up enough
3. Place a oneForZero i.e. sell Token 1 order, and make sure it's executed if tick goes down enough
4. Place two separate orders at different ticks, and make sure only one of them is executed if the fulfillment of the first order makes the second order no longer eligible to be executed
5. Place two separate orders at different ticks, and make sure both are executed if they remain eligible with the tick shift

## test_orderExecute_zeroForOne

In our `setUp` function from Part 1 - we have already added liquidity to the pool, so we don't need to do that process. Also, we've added liquidity symmetrically, so initial tick of our pool is 0 - perfect 1:1 price.

We will first place an order at Tick 100, i.e. a take-profit order to sell Token 0 when price increases. Then, we'll conduct a swap to buy Token 0 (and sell Token 1) such that the tick of the pool increases as Token 0 price increases. As a result, our limit order to sell Token 0 at the increased price should execute as part of that swap inside `afterSwap`.

```Solidity

    function test_orderExecute_zeroForOne() public {
        int24 tick = 100;
        uint256 amount = 1 ether;
        bool zeroForOne = true;

        // Place our order at tick 100 for 10e18 token0 tokens
        int24 tickLower = hook.placeOrder(key, tick, zeroForOne, amount);

        // Do a separate swap from oneForZero to make tick go up
        // Sell 1e18 token1 tokens for token0 tokens
        SwapParams memory params = SwapParams({
            zeroForOne: !zeroForOne,
            amountSpecified: -1 ether,
            sqrtPriceLimitX96: TickMath.MAX_SQRT_PRICE - 1
        });

        PoolSwapTest.TestSettings memory testSettings =
            PoolSwapTest.TestSettings({takeClaims: false, settleUsingBurn: false});

        // Conduct the swap - `afterSwap` should also execute our placed order
        swapRouter.swap(key, params, testSettings, ZERO_BYTES);

        // Check that the order has been executed
        // by ensuring no amount is left to sell in the pending orders
        uint256 pendingTokensForPosition = hook.pendingOrders(key.toId(), tick, zeroForOne);
        assertEq(pendingTokensForPosition, 0);

        // Check that the hook contract has the expected number of token1 tokens ready to redeem
        uint256 positionId = hook.getPositionId(key, tickLower, zeroForOne);
        uint256 claimableOutputTokens = hook.claimableOutputTokens(positionId);
        uint256 hookContractToken1Balance = token1.balanceOf(address(hook));
        assertEq(claimableOutputTokens, hookContractToken1Balance);

        // Ensure we can redeem the token1 tokens
        uint256 originalToken1Balance = token1.balanceOf(address(this));
        hook.redeem(key, tick, zeroForOne, amount);
        uint256 newToken1Balance = token1.balanceOf(address(this));

        assertEq(newToken1Balance - originalToken1Balance, claimableOutputTokens);
    }
```

## test_orderExecute_oneForZero

This one is pretty much the same thing as the last - except in the opposite direction. We place an order to sell Token 1 at tick -100. Then, we do a normal swap to buy Token 1 to make Token 1's price increase i.e. decrease the pool tick - and ensure our order was executed.

```Solidity

    function test_orderExecute_oneForZero() public {
        int24 tick = -100;
        uint256 amount = 10 ether;
        bool zeroForOne = false;

        // Place our order at tick -100 for 10e18 token1 tokens
        int24 tickLower = hook.placeOrder(key, tick, zeroForOne, amount);

        // Do a separate swap from zeroForOne to make tick go down
        // Sell 1e18 token0 tokens for token1 tokens
        SwapParams memory params =
            SwapParams({zeroForOne: true, amountSpecified: -1 ether, sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1});

        PoolSwapTest.TestSettings memory testSettings =
            PoolSwapTest.TestSettings({takeClaims: false, settleUsingBurn: false});

        swapRouter.swap(key, params, testSettings, ZERO_BYTES);

        // Check that the order has been executed
        uint256 tokensLeftToSell = hook.pendingOrders(key.toId(), tick, zeroForOne);
        assertEq(tokensLeftToSell, 0);

        // Check that the hook contract has the expected number of token0 tokens ready to redeem
        uint256 positionId = hook.getPositionId(key, tickLower, zeroForOne);
        uint256 claimableOutputTokens = hook.claimableOutputTokens(positionId);
        uint256 hookContractToken0Balance = token0.balanceOf(address(hook));
        assertEq(claimableOutputTokens, hookContractToken0Balance);

        // Ensure we can redeem the token0 tokens
        uint256 originalToken0Balance = token0.balanceOfSelf();
        hook.redeem(key, tick, zeroForOne, amount);
        uint256 newToken0Balance = token0.balanceOfSelf();

        assertEq(newToken0Balance - originalToken0Balance, claimableOutputTokens);
    }
```

## test_multiple_orderExecute_zeroForOne_onlyOne

In this, we will place two orders - one at tick 0, and one at tick 60. Then, we will conduct a swap such that tick will cross 60 because of the swap, and then the first order will be executed, which will bring the tick back down such that the second order shouldn't be executed.

```Solidity

    function test_multiple_orderExecute_zeroForOne_onlyOne() public {
        PoolSwapTest.TestSettings memory testSettings =
            PoolSwapTest.TestSettings({takeClaims: false, settleUsingBurn: false});

        // Setup two zeroForOne orders at ticks 0 and 60
        uint256 amount = 0.01 ether;

        hook.placeOrder(key, 0, true, amount);
        hook.placeOrder(key, 60, true, amount);

        (, int24 currentTick,,) = manager.getSlot0(key.toId());
        assertEq(currentTick, 0);

        // Do a swap to make tick increase beyond 60
        SwapParams memory params =
            SwapParams({zeroForOne: false, amountSpecified: -0.1 ether, sqrtPriceLimitX96: TickMath.MAX_SQRT_PRICE - 1});

        swapRouter.swap(key, params, testSettings, ZERO_BYTES);

        // Only one order should have been executed
        // because the execution of that order would lower the tick
        // so even though tick increased beyond 60
        // the first order execution will lower it back down
        // so order at tick = 60 will not be executed
        uint256 tokensLeftToSell = hook.pendingOrders(key.toId(), 0, true);
        assertEq(tokensLeftToSell, 0);

        // Order at Tick 60 should still be pending
        tokensLeftToSell = hook.pendingOrders(key.toId(), 60, true);
        assertEq(tokensLeftToSell, amount);
    }
```

## test_multiple_orderExecute_zeroForOne_both

Finally, we will test the case where we place two orders and expect both of them to execute since the swap we conduct increases it enough such that even after the first order is executed the tick doesn't go back down far enough to stop the second order from executing.

```Solidity

    function test_multiple_orderExecute_zeroForOne_both() public {
        PoolSwapTest.TestSettings memory testSettings =
            PoolSwapTest.TestSettings({takeClaims: false, settleUsingBurn: false});

        // Setup two zeroForOne orders at ticks 0 and 60
        uint256 amount = 0.01 ether;

        hook.placeOrder(key, 0, true, amount);
        hook.placeOrder(key, 60, true, amount);

        // Do a swap to make tick increase
        SwapParams memory params =
            SwapParams({zeroForOne: false, amountSpecified: -0.5 ether, sqrtPriceLimitX96: TickMath.MAX_SQRT_PRICE - 1});

        swapRouter.swap(key, params, testSettings, ZERO_BYTES);

        uint256 tokensLeftToSell = hook.pendingOrders(key.toId(), 0, true);
        assertEq(tokensLeftToSell, 0);

        tokensLeftToSell = hook.pendingOrders(key.toId(), 60, true);
        assertEq(tokensLeftToSell, 0);
    }
```

## Running Tests

At this point, try running forge test.

Now you will see all our tests pass - hooray!

```bash
Ran 6 tests for test/TakeProfitsHook.t.sol:TakeProfitsHookTest
[PASS] test_cancelOrder() (gas: 114977)
[PASS] test_multiple_orderExecute_zeroForOne_both() (gas: 564590)
[PASS] test_multiple_orderExecute_zeroForOne_onlyOne() (gas: 498455)
[PASS] test_orderExecute_oneForZero() (gas: 1236894)
[PASS] test_orderExecute_zeroForOne() (gas: 415376)
[PASS] test_placeOrder() (gas: 134052)
Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 5.13ms (5.53ms CPU time)
```

## Conclusion

This module overall was a bump-up in difficulty and complexity from previous lessons. Hopefully you learnt a few different things and at least have an example now of a "more complex" hook design in practice and how you can use hooks to create completely new behaviours inside Uniswap.

Also, hopefully this content makes you excited for the rest of the course as well - this is the first "real" hook design we've done - and we still have many more to go - all designed with the goal to open up your mind to the possibilities that exist within hooks.

As always, if you have any questions - feel free to ask during the workshops, office hours, or on the group. We will try our best to answer you.

Until next time!
