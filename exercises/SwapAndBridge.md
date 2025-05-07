# Periphery: Swap + Bridge

## Intro and Objectives

### Lesson Objectives

- Learn how to build a Router Contract
- Understand situations where Routers make more sense than Hooks
- Build and deploy a router contract that can bridge output tokens from a swap to another chain

### Introduction

So far we've lived up to the name of the course and focused on hooks. This lesson is a bit different - we're not going to do anything related to hooks here - we're going to build a router contract. A router is a periphery contract that isn't a hook, but is the middleman contract between an EOA and the PoolManager contract. The default Swap Router, Modify Liquidity Router, etc. we've been using so far fall under this category.

While this isn't a hook, a custom router can be useful sometimes for certain types of things that you cannot do inside a hook. In today's lesson we're going to integrate with Optimism's native bridging contracts and build our own Swap Router that swaps a token through the pool manager and attempts to bridge the output token from L1 to an Optimism L2, combining the swap + bridge action into a single transaction.

> [!NOTE]
> Some of the code we'll be writing today is related to the specific implementation of Optimism's bridging contracts. The same idea can be repurposed for any alternative chain that has a bridging path available. We will intentionally not focus too much on Optimism-specific code, how it works, or why it is the way it is. I mostly just want to go over how to build a router contract and help you gain an understanding of how to build your own and also how the default ones work.

# Let's Build

## Setting up Foundry

Let's get started with building the hook first. The full code for this project is available at https://github.com/haardikk21/v4-swap-bridge-router.git

First, let's set up Foundry, install v4-periphery and optimisms contracts, set up remappings, and configure `foundry.toml`

```bash
# Initialize a new foundry project
forge init swap-and-bridge-op

# Install v4-periphery
cd swap-and-bridge-op
forge install Uniswap/v4-periphery
forge install ethereum-optimism/optimism

# Place remappings
forge remappings > remappings.txt

# Remove default Counter.sol files
rm ./**/Counter*.sol
```

Now, to configure `foundry.toml` - add the following to its end:

```toml
solc_version = '0.8.26'
evm_version = "cancun"
optimizer_runs = 800
via_ir = false
ffi = true

[rpc_endpoints]
sepolia = "https://rpc.sepolia.org/"
op_sepolia = "https://sepolia.optimism.io"
```

We're using a public RPC URL for ETH Sepolia and OP Sepolia for testing purposes - but feel free to use any that you want.

## Creating `SwapAndBridgeOptimismRouter.sol`

Create a new file named `SwapAndBridgeOptimismRouter.sol` under the `src/` directory and set up the following boilerplate:

`SwapAndBridgeOptimismRouter.sol`

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {Currency, CurrencyLibrary} from "v4-core/types/Currency.sol";
import {CurrencySettler} from "@uniswap/v4-core/test/utils/CurrencySettler.sol";
import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {BalanceDelta} from "v4-core/types/BalanceDelta.sol";
import {IERC20Minimal} from "v4-core/interfaces/external/IERC20Minimal.sol";
import {TransientStateLibrary} from "v4-core/libraries/TransientStateLibrary.sol";

interface IL1StandardBridge {
    function depositETHTo(
        address _to,
        uint32 _minGasLimit,
        bytes calldata _extraData
    ) external payable;
    function depositERC20To(
        address _l1Token,
        address _l2Token,
        address _to,
        uint256 _amount,
        uint32 _minGasLimit,
        bytes calldata _extraData
    ) external;
}

contract SwapAndBridgeOptimismRouter is Ownable {

	using CurrencyLibrary for Currency;
	using CurrencySettler for Currency;
	using TransientStateLibrary for IPoolManager;

	IPoolManager public immutable manager;
    IL1StandardBridge public immutable l1StandardBridge;

	constructor(
        IPoolManager _manager,
        IL1StandardBridge _l1StandardBridge
    ) Ownable(msg.sender) {
        manager = _manager;
        l1StandardBridge = _l1StandardBridge;
    }
}
```

In the above code, we've defined a simple interface for the `L1StandardBridge` contract that Optimism's native bridge uses with the two functions we care about - `depositETHTo` and `depositERC20To`. We've also set up our Router contract with a constructor to save references to the Pool Manager and the L1 Standard Bridge.

Note that we've also made our router a `Ownable` contract. The reason for that is we want to bridge tokens if and only if they're supported by the native bridge since not all tokens are. To do that we will maintain a mapping of token contract addresses on L1 and their equivalent on the L2, and that mapping can only be updated by the owner of the contract. You can think of this also as a DAO controlling which tokens are supported through our router.

Let's define that mapping and a function to update it.

```Solidity
mapping(address l1Token => address l2Token) public l1ToL2TokenAddresses;

// Helper
function addL1ToL2TokenAddress(
    address l1Token,
    address l2Token
) external onlyOwner {
    l1ToL2TokenAddresses[l1Token] = l2Token;
}
```

## Swapping through the Router

For the rest of the router's design we'll follow closely the design of the swap router we have been using throughout the program - `PoolSwapTest.sol`. A user will call `swap` on our router contract, which will then call the `PoolManager.unlock` function, and then conduct the swap inside the `unlockCallback` and settle balances.

Let's start by defining a couple of structs for `CallbackData` and `SwapSettings`.

```Solidity
struct CallbackData {
    address sender;
    SwapSettings settings;
    PoolKey key;
    IPoolManager.SwapParams params;
    bytes hookData;
}

struct SwapSettings {
    bool bridgeTokens;
    address recipientAddress;
}
```

Let's also define a couple errors we can throw if the callback is called by an actor which is not the Pool Manager, and if the token requested to be bridged cannot be bridged.

```Solidity
error CallerNotManager();
error TokenCannotBeBridged();
```

Great! Now to set up the `swap` function itself:

```Solidity
function swap(
    PoolKey memory key,
    IPoolManager.SwapParams memory params,
    SwapSettings memory settings,
    bytes memory hookData
) external payable returns (BalanceDelta delta) {
    // If user requested a bridge of the output tokens
    // we must make sure the output token can be bridged at all
    // otherwise we revert the transaction early
    if (settings.bridgeTokens) {
        Currency l1TokenToBridge = params.zeroForOne
            ? key.currency1
            : key.currency0;

        if (!l1TokenToBridge.isNative()) {
            address l2Token = l1ToL2TokenAddresses[
                Currency.unwrap(l1TokenToBridge)
            ];
            if (l2Token == address(0)) revert TokenCannotBeBridged();
        }
    }

    // Unlock the pool manager which will trigger a callback
    delta = abi.decode(
        manager.unlock(
            abi.encode(
                CallbackData(msg.sender, settings, key, params, hookData)
            )
        ),
        (BalanceDelta)
    );

    // Send any ETH left over to the sender
    uint256 ethBalance = address(this).balance;
    if (ethBalance > 0)
        CurrencyLibrary.NATIVE.transfer(msg.sender, ethBalance);
}
```

Comments in the code should hopefully be explanatory. The only major thing left to do now is write the unlock callback.

```Solidity
function unlockCallback(
    bytes calldata rawData
) external returns (bytes memory) {
    if (msg.sender != address(manager)) revert CallerNotManager();
    CallbackData memory data = abi.decode(rawData, (CallbackData));

    // Call swap on the PM
    BalanceDelta delta = manager.swap(data.key, data.params, data.hookData);

    int256 deltaAfter0 = manager.currencyDelta(
        address(this),
        data.key.currency0
    );
    int256 deltaAfter1 = manager.currencyDelta(
        address(this),
        data.key.currency1
    );

    if (deltaAfter0 < 0) {
        data.key.currency0.settle(
            manager,
            data.sender,
            uint256(-deltaAfter0),
            false
        );
    }

    if (deltaAfter1 < 0) {
        data.key.currency1.settle(
            manager,
            data.sender,
            uint256(-deltaAfter1),
            false
        );
    }

    if (deltaAfter0 > 0) {
        _take(
            data.key.currency0,
            data.settings.recipientAddress,
            uint256(deltaAfter0),
            data.settings.bridgeTokens
        );
    }

    if (deltaAfter1 > 0) {
        _take(
            data.key.currency1,
            data.settings.recipientAddress,
            uint256(deltaAfter1),
            data.settings.bridgeTokens
        );
    }

    return abi.encode(delta);
}
```

Now as for the actual bridging, we'll do that inside the `_take` function. Recall that `take` means taking money from the PM. Depending on if the user specified they wanted to bridge to the L2 or not through the `SwapSettings`, we'll either `take` money from PM and send it directly to the recipient on the L1, or take the money to the router contract first and initiate a bridge transaction for the recipient on the L2.

```Solidity
function _take(
    Currency currency,
    address recipient,
    uint256 amount,
    bool bridgeToOptimism
) internal {
    // If not bridging, just send the tokens to the swapper
    if (!bridgeToOptimism) {
        currency.take(manager, recipient, amount, false);
    } else {
        // If we are bridging, take tokens to the router and then bridge to the recipient address on the L2
        currency.take(manager, address(this), amount, false);

        if (currency.isNative()) {
            l1StandardBridge.depositETHTo{value: amount}(recipient, 0, "");
        } else {
            address l1Token = Currency.unwrap(currency);
            address l2Token = l1ToL2TokenAddresses[l1Token];

            IERC20Minimal(l1Token).approve(
                address(l1StandardBridge),
                amount
            );
            l1StandardBridge.depositERC20To(
                l1Token,
                l2Token,
                recipient,
                amount,
                0,
                ""
            );
        }
    }
}
```

With this, we're basically done. One final little thing is our contract needs to define a `receive()` function since swaps through the PM have the ability to send our router contract some `ETH` back - but our contract cannot accept `ETH` transfers until we define a `receive` function.

```Solidity
receive() external payable {}
```

Perfect! And we're done with the contract!

## Creating test Files

Create a new file named `TestSwapAndBridgeOptimismRouter.t.sol` under the `test/` directory.

Since we're testing a bridging action which involves a trigger transaction on the L1 which is then processed by the Optimism network offchain leading to a transaction on the L2, we cannot test the full flow since we'll be working off of a local fork of the network. Instead in our tests we can simply verify that all the events that are expected to be emitted from the `L1StandardBridge` contract and other related Optimism contracts are being emitted, and trust that the network will do its thing.

For testing on Sepolia, Optimism has an ERC-20 called `OUTb` (Optimism Useless Token Bridged) that has an infinite faucet supply and supports the native bridge. We'll set up a `ETH/OUTb pool` and test against that. The tests aren't too complicated here (but do have some Optimism-specific code), so there are relevant comments throughout the code to help make sense of it.

`TestSwapAndBridgeOptimismRouter.t.sol`

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Test, Vm} from "forge-std/Test.sol";
import {Deployers} from "@uniswap/v4-core/test/utils/Deployers.sol";
import {MockERC20} from "solmate/src/test/utils/mocks/MockERC20.sol";
import {PoolManager} from "v4-core/PoolManager.sol";
import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {Currency, CurrencyLibrary} from "v4-core/types/Currency.sol";
import {PoolId, PoolIdLibrary} from "v4-core/types/PoolId.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {Hooks} from "v4-core/libraries/Hooks.sol";
import {IHooks} from "v4-core/interfaces/IHooks.sol";
import {SwapAndBridgeOptimismRouter, IL1StandardBridge} from "../src/SwapAndBridgeOptimismRouter.sol";
import {TickMath} from "v4-core/libraries/TickMath.sol";

interface IOUTbToken {
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
    function faucet() external;
}

contract TestSwapAndBridgeOptimismRouter is Test, Deployers {
    using CurrencyLibrary for Currency;
    using PoolIdLibrary for PoolKey;

    /*//////////////////////////////////////////////////////////////
                                 EVENTS
    These are events from L1StandardBridge and CrossDomainMessenger
    //////////////////////////////////////////////////////////////*/

    event ETHDepositInitiated(
        address indexed from,
        address indexed to,
        uint256 amount,
        bytes extraData
    );

    event ERC20DepositInitiated(
        address indexed l1Token,
        address indexed l2Token,
        address indexed from,
        address to,
        uint256 amount,
        bytes extraData
    );

    event ETHBridgeInitiated(
        address indexed from,
        address indexed to,
        uint256 amount,
        bytes extraData
    );

    event ERC20BridgeInitiated(
        address indexed localToken,
        address indexed remoteToken,
        address indexed from,
        address to,
        uint256 amount,
        bytes extraData
    );

    event SentMessage(
        address indexed target,
        address sender,
        bytes message,
        uint256 messageNonce,
        uint256 gasLimit
    );

    event SentMessageExtension1(address indexed sender, uint256 value);

    /*//////////////////////////////////////////////////////////////
                            TEST STORAGE
    //////////////////////////////////////////////////////////////*/

    uint256 sepoliaForkId = vm.createFork("https://rpc.sepolia.org/");

    SwapAndBridgeOptimismRouter poolSwapAndBridgeOptimism;

    // OUTb = Optimism Useless Token Bridged (ETH Sepolia and OP Sepolia addresses)
    IOUTbToken OUTbL1Token =
        IOUTbToken(0x12608ff9dac79d8443F17A4d39D93317BAD026Aa);
    IOUTbToken OUTbL2Token =
        IOUTbToken(0x7c6b91D9Be155A6Db01f749217d76fF02A7227F2);

    // L1 Standard Bridge on ETH Sepolia
    IL1StandardBridge public constant l1StandardBridge =
        IL1StandardBridge(0xFBb0621E0B23b5478B630BD55a5f21f67730B0F1);

    // Cross Domain Messenger L2 Contract Address
    address public constant l2CrossDomainMessenger =
        0x4200000000000000000000000000000000000010;

    /*//////////////////////////////////////////////////////////////
                            TEST SETUP
    //////////////////////////////////////////////////////////////*/

    function setUp() public {
        vm.selectFork(sepoliaForkId);
        vm.deal(address(this), 500 ether);

        // Deploy manager and routers
        deployFreshManagerAndRouters();
        poolSwapAndBridgeOptimism = new SwapAndBridgeOptimismRouter(
            manager,
            l1StandardBridge
        );

        // Get some OUTb tokens on L1 and approve the routers to use it
        OUTbL1Token.faucet();
        OUTbL1Token.approve(
            address(poolSwapAndBridgeOptimism),
            type(uint256).max
        );
        OUTbL1Token.approve(address(modifyLiquidityRouter), type(uint256).max);

        // Create the OUTb token mapping on the periphery contract
        poolSwapAndBridgeOptimism.addL1ToL2TokenAddress(
            address(OUTbL1Token),
            address(OUTbL2Token)
        );

        // Deploy an ETH <> OUTb pool and add some liquidity there
        (key, ) = initPool(
            CurrencyLibrary.NATIVE,
            Currency.wrap(address(OUTbL1Token)),
            IHooks(address(0)),
            3000,
            SQRT_PRICE_1_1,
            ZERO_BYTES
        );
        modifyLiquidityRouter.modifyLiquidity{value: 1 ether}(
            key,
            IPoolManager.ModifyLiquidityParams({
                tickLower: -60,
                tickUpper: 60,
                liquidityDelta: 10 ether,
                salt: bytes32(0)
            }),
            ZERO_BYTES
        );
    }

    /**
     * As long as we are running on a fork, we cannot check the OP Sepolia side of things
     *     So we will only test based on the events being output by the contract
     *     A separate script file exists which tests on actual Sepolia Testnet and OP Sepolia
     */

    /*//////////////////////////////////////////////////////////////
                            TEST SWAP ETH FOR OUTb
                            WITH BRIDGING TO OP
                            RECIPIENT = SENDER
    //////////////////////////////////////////////////////////////*/

    function test_swapETHForOUTb_bridgeTokensToOptimism_recipientSameAsSender()
        public
    {
        vm.expectEmit(true, true, true, false);
        emit ERC20DepositInitiated(
            address(OUTbL1Token),
            address(OUTbL2Token),
            address(poolSwapAndBridgeOptimism),
            address(this),
            0,
            ZERO_BYTES
        );

        vm.expectEmit(true, true, true, false);
        emit ERC20BridgeInitiated(
            address(OUTbL1Token),
            address(OUTbL2Token),
            address(poolSwapAndBridgeOptimism),
            address(this),
            0,
            ZERO_BYTES
        );

        vm.expectEmit(true, false, false, false);
        emit SentMessage(l2CrossDomainMessenger, address(0), ZERO_BYTES, 0, 0);

        vm.expectEmit(true, false, false, false);
        emit SentMessageExtension1(address(l1StandardBridge), 0);

        poolSwapAndBridgeOptimism.swap{value: 0.001 ether}(
            key,
            IPoolManager.SwapParams({
                zeroForOne: true,
                amountSpecified: -0.001 ether,
                sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
            }),
            SwapAndBridgeOptimismRouter.SwapSettings({
                bridgeTokens: true,
                recipientAddress: address(this)
            }),
            ZERO_BYTES
        );
    }

    /*//////////////////////////////////////////////////////////////
                            TEST SWAP ETH FOR OUTb
                            WITH BRIDGING TO OP
                            RECIPIENT != SENDER
    //////////////////////////////////////////////////////////////*/

    function test_swapETHForOUTb_bridgeTokensToOptimism_receipientNotSameAsSender()
        public
    {
        address recipientAddress = address(0x1);

        vm.expectEmit(true, true, true, false);
        emit ERC20DepositInitiated(
            address(OUTbL1Token),
            address(OUTbL2Token),
            address(poolSwapAndBridgeOptimism),
            recipientAddress,
            0,
            ZERO_BYTES
        );

        vm.expectEmit(true, true, true, false);
        emit ERC20BridgeInitiated(
            address(OUTbL1Token),
            address(OUTbL2Token),
            address(poolSwapAndBridgeOptimism),
            recipientAddress,
            0,
            ZERO_BYTES
        );

        vm.expectEmit(true, false, false, false);
        emit SentMessage(l2CrossDomainMessenger, address(0), ZERO_BYTES, 0, 0);

        vm.expectEmit(true, false, false, false);
        emit SentMessageExtension1(address(l1StandardBridge), 0);

        poolSwapAndBridgeOptimism.swap{value: 0.001 ether}(
            key,
            IPoolManager.SwapParams({
                zeroForOne: true,
                amountSpecified: -0.001 ether,
                sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
            }),
            SwapAndBridgeOptimismRouter.SwapSettings({
                bridgeTokens: true,
                recipientAddress: recipientAddress
            }),
            ZERO_BYTES
        );
    }

    /*//////////////////////////////////////////////////////////////
                            TEST SWAP OUTb FOR ETH
                            WITH BRIDGING TO OP
                            RECIPIENT = SENDER
    //////////////////////////////////////////////////////////////*/

    function test_swapOUTbForETH_bridgeTokensToOptimism_recipientSameAsSender()
        public
    {
        vm.expectEmit(true, true, false, false);
        emit ETHDepositInitiated(
            address(poolSwapAndBridgeOptimism),
            address(this),
            0,
            ZERO_BYTES
        );

        vm.expectEmit(true, true, false, false);
        emit ETHBridgeInitiated(
            address(poolSwapAndBridgeOptimism),
            address(this),
            0,
            ZERO_BYTES
        );

        vm.expectEmit(true, false, false, false);
        emit SentMessage(l2CrossDomainMessenger, address(0), ZERO_BYTES, 0, 0);

        vm.expectEmit(true, false, false, false);
        emit SentMessageExtension1(address(l1StandardBridge), 0);

        poolSwapAndBridgeOptimism.swap(
            key,
            IPoolManager.SwapParams({
                zeroForOne: false,
                amountSpecified: -0.001 ether,
                sqrtPriceLimitX96: TickMath.MAX_SQRT_PRICE - 1
            }),
            SwapAndBridgeOptimismRouter.SwapSettings({
                bridgeTokens: true,
                recipientAddress: address(this)
            }),
            ZERO_BYTES
        );
    }

    /*//////////////////////////////////////////////////////////////
                            TEST SWAP OUTb FOR ETH
                            WITH BRIDGING TO OP
                            RECIPIENT != SENDER
    //////////////////////////////////////////////////////////////*/
    function test_swapOUTbForETH_bridgeTokensToOptimism_recipientNotSameAsSender()
        public
    {
        address recipientAddress = address(0x1);

        vm.expectEmit(true, true, false, false);
        emit ETHDepositInitiated(
            address(poolSwapAndBridgeOptimism),
            recipientAddress,
            0,
            ZERO_BYTES
        );

        vm.expectEmit(true, true, false, false);
        emit ETHBridgeInitiated(
            address(poolSwapAndBridgeOptimism),
            recipientAddress,
            0,
            ZERO_BYTES
        );

        vm.expectEmit(true, false, false, false);
        emit SentMessage(l2CrossDomainMessenger, address(0), ZERO_BYTES, 0, 0);

        vm.expectEmit(true, false, false, false);
        emit SentMessageExtension1(address(l1StandardBridge), 0);

        poolSwapAndBridgeOptimism.swap(
            key,
            IPoolManager.SwapParams({
                zeroForOne: false,
                amountSpecified: -0.001 ether,
                sqrtPriceLimitX96: TickMath.MAX_SQRT_PRICE - 1
            }),
            SwapAndBridgeOptimismRouter.SwapSettings({
                bridgeTokens: true,
                recipientAddress: recipientAddress
            }),
            ZERO_BYTES
        );
    }

    /*//////////////////////////////////////////////////////////////
                            TEST SWAP ETH FOR OUTb
                            WITHOUT BRIDGING
    //////////////////////////////////////////////////////////////*/
    function test_swapETHForOUTb_dontBridgeTokens() public {
        uint256 ethBalanceBefore = address(this).balance;
        uint256 OUTbBalanceBefore = OUTbL1Token.balanceOf(address(this));

        poolSwapAndBridgeOptimism.swap{value: 0.001 ether}(
            key,
            IPoolManager.SwapParams({
                zeroForOne: true,
                amountSpecified: -0.001 ether,
                sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
            }),
            SwapAndBridgeOptimismRouter.SwapSettings({
                bridgeTokens: false,
                recipientAddress: address(this)
            }),
            ZERO_BYTES
        );

        uint256 ethBalanceAfter = address(this).balance;
        uint256 OUTbBalanceAfter = OUTbL1Token.balanceOf(address(this));

        assertEq(ethBalanceBefore - ethBalanceAfter, 0.001 ether);
        assertGt(OUTbBalanceAfter, OUTbBalanceBefore);
    }

    /*//////////////////////////////////////////////////////////////
                            TEST SWAP OUTb FOR ETH
                            WITHOUT BRIDGING
    //////////////////////////////////////////////////////////////*/

    function test_swapOUTbForETH_dontBridgeTokens() public {
        uint256 ethBalanceBefore = address(this).balance;
        uint256 OUTbBalanceBefore = OUTbL1Token.balanceOf(address(this));

        poolSwapAndBridgeOptimism.swap(
            key,
            IPoolManager.SwapParams({
                zeroForOne: false,
                amountSpecified: -0.001 ether,
                sqrtPriceLimitX96: TickMath.MAX_SQRT_PRICE - 1
            }),
            SwapAndBridgeOptimismRouter.SwapSettings({
                bridgeTokens: false,
                recipientAddress: address(this)
            }),
            ZERO_BYTES
        );

        uint256 ethBalanceAfter = address(this).balance;
        uint256 OUTbBalanceAfter = OUTbL1Token.balanceOf(address(this));

        assertGt(ethBalanceAfter, ethBalanceBefore);
        assertEq(OUTbBalanceBefore - OUTbBalanceAfter, 0.001 ether);
    }
}
```

## Conclusion

This was a fairly short module - but goes to show how first of all building a router isn't really that complicated, and also building a router can enable you to do things that'd be much harder to do inside a hook for certain cases.

Also, in the Github repo there is a Foundry script that you can use to test manually against the real Sepolia network and see the bridge transaction occur on Optimism Sepolia as well after a while - but it's not too important for the purposes of this lesson.
