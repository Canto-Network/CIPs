# Onboarding module

## CPIP-002

| CPIP  | Title             | Author                                                       | Status | Type            | Category | Created    |
| ---- | ----------------- | ------------------------------------------------------------ | ------ | --------------- | -------- | ---------- |
| 002  | Onboarding module | Byungchul Park(@poorph), Hyunwoo Lee(@zsystm), Dongsam Byun(@dongsam) | Draft  | Standards Track | Core     | 2023-05-27 |

# Simple Summary

The module which implements IBC middleware for users who don’t have even a small amount of Canto tokens for initial gas spending.

# Abstract

We introduce a new module called `onboarding` to help users outside of Canto onboard seamlessly.

The module automatically swaps a portion of the assets being transferred to Canto network via IBC transfer for Canto without the need for a manual process, and converts the remaining assets to ERC20 tokens on Canto.

# Specification

When users transfer assets to the Canto network through Gravity Bridge, the IBC transfer automatically triggers swap and conversion to Canto ERC20 via IBC middleware. These actions are triggered only when transferred through a whitelisted channel.

## Procedure

- User transfers assets to the Canto network through Gravity Bridge
- Check recipient address's Canto balance
- If the balance is less than the minimum threshold (e.g. `4canto`), swap the assets to Canto
- If the remaining assets are registered as a token pair, convert them to ERC20.

## Module Parameters

```go
message Params {
  bool enable_onboarding = 1;

  string auto_swap_threshold = 3 [
    (gogoproto.customtype) = "github.com/cosmos/cosmos-sdk/types.Int",
    (gogoproto.nullable) = false
  ];

  repeated string whitelisted_channels = 4;
}
```

- **`EnableOnboarding`:** If this value is `false`, then it will disable the auto swap and convert. (default value: `true`)
- **`AutoSwapThreshold`:** the threshold of the amount of canto to be swapped. When the balance of canto is less than the threshold, the auto swap will be triggered. (default value: `4 canto`)
- **`WhitelistedChannels`:** The list of channels that will be whitelisted for the auto swap and convert. When the channel is not in the list, the auto swap and convert will not be triggered. (default value: `["channel-0"]` which is a channel for Gravity Bridge)

## OnRecvPacket

The `onboarding` module implements an IBC Middleware in order to swap and convert IBC transferred assets to Canto and ERC20 tokens with `Keeper.OnRecvPacket` callback.

1. A user performs an IBC transfer to the Canto network. This is done using a `FungibleTokenPacket` IBC packet.
2. Check that the onboarding conditions are met and skip to the next middleware if any condition is not satisfied:
   1. onboarding is enabled globally
   2. channel is authorized
   3. the recipient account is not a module account
3. Check the recipient's Canto balance and if the balance is less than the `AutoSwapThreshold`, swap the assets to Canto. Amount of the swapped Canto is always equal to the `AutoSwapThreshold` and the price is determined by the liquidity pool.
4. Check if the transferred asset is registered in the `x/erc20` module as a ERC20 token pair and the token pair is enabled. If so, convert the remaining assets to ERC20 tokens.

```go
func (k Keeper) OnRecvPacket(
	ctx sdk.Context,
	packet channeltypes.Packet,
	ack exported.Acknowledgement,
) exported.Acknowledgement {
	// It always returns original ACK if the packet was a ICS20 transfer.
	// Which means even if the swap or conversion fails, it does not revert IBC transfer.
	// The asset transferred to the Canto network will still remain in the Canto network.
	params := k.GetParams(ctx)
	if !params.EnableOnboarding {
		return ack
	}
	// check source channel is in the whitelist channels
	var found bool
	for _, s := range params.WhitelistedChannels {
		if s == packet.DestinationChannel {
			found = true
		}
	}
	if !found {
		return ack
	}
	// Get recipient addresses in `canto1` and the original bech32 format
	_, recipient, senderBech32, recipientBech32, err := ibc.GetTransferSenderRecipient(packet)
	if err != nil {
		return channeltypes.NewErrorAcknowledgement(err.Error())
	}
	account := k.accountKeeper.GetAccount(ctx, recipient)
	// onboarding is not supported for module accounts
	if _, isModuleAccount := account.(authtypes.ModuleAccountI); isModuleAccount {
		return ack
	}
	standardDenom := k.coinswapKeeper.GetStandardDenom(ctx)
	var data transfertypes.FungibleTokenPacketData
	if err = transfertypes.ModuleCdc.UnmarshalJSON(packet.GetData(), &data); err != nil {
		// NOTE: shouldn't happen as the packet has already
		// been decoded on ICS20 transfer logic
		err = errorsmod.Wrapf(types.ErrInvalidType, "cannot unmarshal ICS-20 transfer packet data")
		return channeltypes.NewErrorAcknowledgement(err.Error())
	}

	// parse the transferred denom
	transferredCoin := ibc.GetReceivedCoin(
		packet.SourcePort, packet.SourceChannel,
		packet.DestinationPort, packet.DestinationChannel,
		data.Denom, data.Amount,
	)
	autoSwapThreshold := k.GetParams(ctx).AutoSwapThreshold
	swapCoins := sdk.NewCoin(standardDenom, autoSwapThreshold)
	standardCoinBalance := k.bankKeeper.SpendableCoins(ctx, recipient).AmountOf(standardDenom)
	swappedAmount := sdk.ZeroInt()
	// Swap for users who have less canto than the autoSwapThreshold.
	if standardCoinBalance.LT(autoSwapThreshold) {
		swappedAmount, err = k.coinswapKeeper.TradeInputForExactOutput(ctx, coinswaptypes.Input{Coin: transferredCoin, Address: recipient.String()}, coinswaptypes.Output{Coin: swapCoins, Address: recipient.String()})
		if err != nil {
			// no-op: proceed with the remaining logic regardless of 
      // whether the swap is successful or not.
		}		
	}

	// convert coins to ERC20 token if denom is registered in erc20 module.
	pairID := k.erc20Keeper.GetTokenPairID(ctx, transferredCoin.Denom)
	if len(pairID) == 0 {
		// short-circuit: if the denom is not registered, conversion will fail
		// so we can continue with the rest of the stack
		return ack
	}
	pair, _ := k.erc20Keeper.GetTokenPair(ctx, pairID)
	if !pair.Enabled {
		// no-op: continue with the rest of the stack without conversion
		return ack
	}
	convertCoin := sdk.NewCoin(transferredCoin.Denom, transferredCoin.Amount.Sub(swappedAmount))
	// Build MsgConvertCoin, from recipient to recipient since IBC transfer already occurred
	convertMsg := erc20types.NewMsgConvertCoin(convertCoin, common.BytesToAddress(recipient.Bytes()), recipient)

	// NOTE: we don't use ValidateBasic the msg since we've already validated
	// the ICS20 packet data
	// Use MsgConvertCoin to convert the Cosmos Coin to an ERC20
	if _, err = k.erc20Keeper.ConvertCoin(sdk.WrapSDKContext(ctx), convertMsg); err != nil {
		return ack
	}

	// return original acknowledgement
	return ack
}
```

It is possible that the IBC transaction fails in any point of the stack execution and in that case the onboarding will not be triggered by the transaction, as it will rollback to the previous state. However, the onboarding process is non-atomic, meaning that even if the swap or conversion fails, it does not revert IBC transfer and the asset transferred to the Canto network will still remain in the Canto network.

## Swap

For swap, we use a forked version of IRISNET's [Coinswap module v1.6](https://github.com/irisnet/irismod/tree/v1.6.0/modules/coinswap), which includes some modifications.

IRISNET's Coinswap module uses an AMM-based swap. This means that onboarding swaps will be handled by AMM also. However, there are some modifications:

- Only token pairs on the whitelist can be created as a pool.
  - Pool creation fails if the token pair is not on the whitelist.
  - Initial whitelist: `Canto/USDC.grv`, `Canto/USDT.grv`, `Canto/ETH.grv`
- There is a limit on the number of Canto tokens for each pool.
  - Deposits will fail if the amount of Canto for the pool exceeds 10,000 Canto.
- Double swaps are disabled.

For risk management purposes, a swap will fail if the input coin amount exceeds a pre-defined limit (10 USDC, 10 USDT, 0.01 ETH) or if the swap amount limit is not defined.

```go
// Only those IBC denom tokens are allowed to convert to Canto.
const (
	UsdcIBCDenom = "ibc/17CD484EE7D9723B847D95015FA3EBD1572FD13BC84FB838F55B18A57450F25B"
	UsdtIBCDenom = "ibc/4F6A2DEFEA52CD8D90966ADCB2BD0593D3993AB0DF7F6AEB3EFD6167D79237B0"
	EthIBCDenom  = "ibc/DC186CA7A8C009B43774EBDC825C935CABA9743504CE6037507E6E5CCE12858A"
)

// Default parameters
var (
	DefaultFee                    = sdk.NewDecWithPrec(0, 0)
	DefaultPoolCreationFee        = sdk.NewInt64Coin(sdk.DefaultBondDenom, 0)
	DefaultTaxRate                = sdk.NewDecWithPrec(0, 0)
	// Limit the number of canto tokens for each pool.
	DefaultMaxStandardCoinPerPool = sdk.NewIntWithDecimal(10000, 18)
	// Pre-defined limits of swap amount for risk management purposes.
	DefaultMaxSwapAmount          = sdk.NewCoins(
		sdk.NewCoin(UsdcIBCDenom, sdk.NewIntWithDecimal(10, 6)),
		sdk.NewCoin(UsdtIBCDenom, sdk.NewIntWithDecimal(10, 6)),
		sdk.NewCoin(EthIBCDenom, sdk.NewIntWithDecimal(1, 17)),
	)
)
```

## Middleware ordering

The IBC middleware adds custom logic between the core IBC and the underlying application. Middlewares are implemented as stacks so that applications can define multiple layers of custom behavior. Function calls from the IBC core to the application travel from the top-level middleware to the bottom middleware, and then to the application.

For Canto the middleware stack ordering is defined as follows (from top to bottom):

1. IBC Transfer
2. Recovery Middleware
3. Onboarding Middleware

```go
// app.go
// create IBC module from bottom to top of stack
var transferStack porttypes.IBCModule

transferStack = transfer.NewIBCModule(app.TransferKeeper)
transferStack = recovery.NewIBCMiddleware(*app.RecoveryKeeper, transferStack)
transferStack = onboarding.NewIBCMiddleware(*app.OnboardingKeeper, transferStack)
```

Each module implements their own custom logic in the packet callback `OnRecvPacket`. When a packet arrives from the IBC core, the IBC transfer will be executed first, followed by an attempted recovery, and finally the onboarding will be executed
