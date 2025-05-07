# MEV: Async Swaps

## Intro and Objectives

### Lesson Objectives

- Understand sandwich attacks and how they work
- Analyze a real onchain sandwich attack to see it in action
- Understand how delaying swap execution until a later point in time can help mitigate sandwich attacks

### Introduction

Today we're going to go over a short lesson that introduces another use case of `NoOp` hooks that can help mitigate some toxic MEV that AMM pools face. Particularly we're going to talk about sandwich attacks during swaps, and how an asynchronous swap design can help mitigate certain types of sandwich attacks.

## Sandwich Attacks

Sandwiching is a type of toxic MEV that exists when an MEV searcher attempts to profit from the price volatility of an asset created due to a swapper's swap request. The name "sandwich" comes from the fact that to do so the MEV searcher typically would place one transaction right before the user's swap, and one transaction right after the user's swap - therefore "sandwiching" the user's transaction with two of their own.

To understand conceptually how this works let's look at a hypothetical example, and then we'll look at a real one onchain.

Assume a pool exists for ETH/TOKEN. Let's say that currently 1 ETH = 1000 TOKEN. Also let's assume this pool doesn't have a lot of liquidity available (not hundreds of millions of dollars), so relatively "small" swaps can still cause decent slippage.

Alice wants to sell 0.5 ETH to buy 500 TOKEN. The minimum she's willing to accept for her 0.5 ETH is 450 TOKEN. She submits a swap transaction onchain for her 0.5 ETH with a slippage value of 450 TOKEN set.

An MEV searcher sees this transaction and realizes they can make a profit. They construct two transactions - a frontrun and a backrun.

In the frontrun transaction (the txn right before Alice's), the MEV searcher spends some ETH buying up some amount of TOKEN. By doing so, they increase the price of TOKEN relative to ETH. They calculate the amount of ETH they sell based on exactly what is required to make ETH/TOKEN price shift just enough that Alice's swap in the next transaction will just barely give her 450 TOKEN which was her minimum slippage amount.

Then, Alice's transaction is processed, and she spends 0.5 ETH to get back ~450 TOKEN.

At this point, there have been two transactions in the pool in a row to buy TOKEN, which have increased the relative price of TOKEN to ETH. The MEV searcher's backrun transaction is now executed, in which the searcher will sell back the TOKEN they purchased for ETH. Since TOKEN is now trading at a higher ETH price than it was when they bought it, the searcher will get back slightly more ETH than they spent previously, therefore making a profit.

Now, let's see this in action onchain. We'll inspect a transaction from an MEV searcher famous for their sandwich attacks - `jaredfromsubway.eth` - who has made tens of millions in profit from MEV searching over the last couple years or so.

The three transactions are as follows:

1. [Jared's Frontrun](https://etherscan.io/tx/0x2d268cbbe55124c0e52a6ec37512d99e941cad38abd7a1876bf4d2c3a1b8d423)
2. [User's Swap](https://etherscan.io/tx/0x0f80c7d7d802ee5aa1271010abb610a67aefa54a399e9cb4126d66a2d46cf286)
3. [Jared's Backrun](https://etherscan.io/tx/0x6680cf9c3abca0a999eb9e6ce7f4d05969e30b904455d48c73fe24778800ee57)

The user in this example wanted to exchange `0.25 WETH` to buy some amount of the `XPAYPAL` token.

Jared noticed this transaction in the mempool, and sent a frontrun transaction to buy `11,191,536 XPAYPAL` for `0.1185 WETH`. This amount was chosen through careful calculation offchain to increase the relative price of the `XPAYPAL` token just enough to meet the user's slippage requirement.

Then the user's swap transaction was executed where they spent `0.25 WETH` to purchase `20,913,009 XPAYPAL` in return. At this point, two transactions have been conducted which increased the relative price of `XPAYPAL` in the pool.

Jared's backrun transaction then executed to sell back the `11,191,536 XPAYPAL` tokens to the pool, exchanging it for `0.1353 WETH`. This means that Jared made 14% (0.0168 WETH / $30 at the time) more WETH than they spent.

> [!NOTE]
> The 0.0168 WETH is revenue, not profit. To calculate profit we need to consider Jared's cost of tipping block proposers to include his transaction bundle in their block. Once we consider that, Jared only made ~0.003 USD in profit. But Jared has done millions of transactions - some good some bad - and overall is one of the most well known sandwich bots out there.

## Mitigating Sandwich Attacks

Sandwich attacks suck for actual users. Considering the above example, Jared made a teeny little bit of profit and the user conducting the swap got absolutely the minimum amount they indicated based on their slippage requirement (which usually is auto-selected by frontends) causing them to lose out on $20-30 in the swap had the sandwich not existed at all.

This type of attack however happens much more often than you think. If you've ever traded on an AMM, knowingly, or unknowingly, you've likely also been sandwiched at some point.

Before we think about mitigating these attacks through an async swap design, let's understand why an asynchronous design works at all.

Note that a sandwich works by increasing the relative price of a token in the frontrun transaction, then having the user's transaction further increase the price of that token, and then immediately sell the tokens bought in the backrun transaction. By design, therefore, sandwich attacks only work if the bundle can be processed in the specific order of Frontrun → Swap → Backrun. Sandwich attacks don't really occur across multiple blocks since the opportunity is lost by that point anyway.

Therefore, what we can do is to intentionally make swaps happen across blocks instead of happening atomically. Maybe not every single swap, but at least some of the "large" swaps (large defined by % price movement expected since it varies for different pools).

So for example instead of letting the user swap their `0.25 WETH` for the `XPAYPAL` token immediately - what if they sent `0.25 ETH` to the contract, but didn't get their `XPAYPAL` token back until a later point, maybe in the next block or something. But it's important to not have the exact execution time be deterministic, because if it's highly predictable for all users in a given pool then sandwich-like attacks can be constructed in the same way as before.

So the goal is to then lock up the input token in the contract, and not execute the swap until some point later in time with some randomness thrown into the mix. The unpredictable nature of this type of design means sandwich attacks will be unable to be constructed since MEV searchers won't know when exactly the price shift will happen, especially if those async transactions themselves are going through MEV Protected RPC endpoints instead of the public mempool.

## Mechanism Design

So our goal is a bit clearer now:

When we see a swap transaction, if it's considered to be a "large" enough swap, we should not execute the swap immediately and instead lock up the input tokens and have it be executed at a later point in time randomly (with reasonable upper limits).

To do so, we need to answer a few questions:

1. What is considered to be a large swap?
2. How do we lock up input tokens?
3. When should a "paused" swap be executed, exactly?
4. Who is conducting the transaction to execute the paused swap? How are they being paid back for their gas costs?

For (1), transactions most often targeted to be sandwich attacked are ones with a "high" slippage tolerance where MEV bots can stand to profit from sandwiching it. High slippage tolerances generally exist either in low-liquidity pools (memecoins, small cap projects, newly launched tokens, etc) or when a large swap is taking place. A simple onchain algorithm to check if the slippage is above a certain threshold is good enough for a proof of concept here.

For (2), we can do a Return Delta in beforeSwap to have the hook take custody of the input tokens and return no output tokens back to the user,.

For (3), this should ideally be an offchain component. For a proof of concept a centralized version is probably fine, but a decentralized network could also be built. The job of this offchain processor is to basically listen for when a swap is paused by the hook due to it being considered large, and assign it some random execution timestamp in the near future. Later, at that time, it should conduct a transaction (ideally through an MEV protected RPC) to instruct the hook to resume that swap transaction.

For (4), the offchain processor is the one executing the paused transactions - but how they are able to sustain themselves is something to think about. Something that can be done here is basically charging some fees on the hook level for "saving" users from toxic MEV. For example, if you identify a transaction that stood to lose $50 due to it being sandwichable, and you were able to prevent that from happening by pausing the transaction, it's fair to say you want to charge a $5 fee for your work but still give the swapper back $45 more than they would've gotten otherwise. This $5 fee should hopefully cover the offchain processor's gas costs and maybe then some. The exact amount of fees you want to charge should maybe be a certain % of the expected loss the user would have taken had they been sandwiched.

## Conclusion

While this is a relatively simple design, it shows how a "simple" `NoOp` hook idea can help mitigate toxic MEV for the everyday user. Though the user experience of this can be a bit confusing if not designed carefully (WHERE ARE MY TOKENS???) - I'm sure you can find a way to display the information in a way that makes sense on a frontend.

Anyway, hopefully you enjoyed this. If you have any questions feel free to post on the Discord or join my office hours. Until then, ciao!
