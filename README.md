# Contract Audit Report

## Scope

Target contract: `WorldcoreBNBLaunch`  
Source repository: `Leviako2025/Token-Audit-For-Report`  
Audit file reviewed: `contracts_WorldcoreBNBLaunch_Version2.sol.txt`

## Executive Summary

The reviewed token contract is a standard ERC-20 with taxation, anti-bot limits, vesting, and liquidity-lock logic. The implementation is mostly structured as a launch token, but it contains a material bug in the vesting release flow and a few logic issues that could cause funds to be routed incorrectly or blocks to be bypassed during the launch phase.

Overall assessment: Moderate risk. The most important issue is a vesting payout bug that sends released tokens to the owner instead of the vesting beneficiary, which can misroute team allocations and break trust assumptions.

## Findings

| Severity | Finding | Status |
| --- | --- | --- |
| High | Vesting release sends funds to the owner instead of the beneficiary | Confirmed |
| Medium | Liquidity lock/release logic is tied to the owner rather than a dedicated liquidity wallet/locker | Confirmed |
| Medium | Anti-bot safeguards are only partially enforced because the pair address may be unset or not validated consistently | Confirmed |
| Low | Tax and transfer logic is vulnerable to misunderstanding if the project intends to keep a live pair but never sets it | Design risk |

## 1) High - Vesting release sends funds to the owner

### Details

The vesting schedule creation function transfers tokens to the beneficiary:

```solidity
_transfer(msg.sender, beneficiary, amount);
```

However, when the vesting is released, the token is sent to the `owner()` instead of the beneficiary:

```solidity
schedule.released += releasable;
_transfer(beneficiary, owner(), releasable);
```

This is the opposite of the normal vesting pattern and breaks the intended model where a beneficiary receives their vested allocation. In practical terms, the contract can drain the beneficiary's vesting balance into the owner address while the vesting record still marks the amount as released.

### Impact

- Team allocations can be incorrectly routed.
- Beneficiaries may never receive funds that were scheduled to them.
- Ownership/ treasury accounting becomes unreliable.
- The violation is material enough to require immediate fix before launch or fund distribution.

### Recommendation

Change the release logic so the vested amount goes to the beneficiary and only the owner can trigger release:

```solidity
schedule.released += releasable;
_transfer(address(this), beneficiary, releasable);
```

or, even better, keep tokens in a dedicated vesting wallet and transfer from that wallet to the beneficiary.

## 2) Medium - Liquidity lock logic is mis-scoped and owner centered

### Details

Liquidity lock uses the owner as the key and then transfers from the owner to the contract:

```solidity
liquidityLocks[owner()] = LiquidityLock({ ... });
_transfer(msg.sender, address(this), amount);
```

The release flow then sends funds back to the owner:

```solidity
_transfer(address(this), owner(), releasable);
```

This means the liquidity lock is effectively a lock on the owner's balance, not a neutral or project-level liquidity vault. That is acceptable only if the owner is intentionally the liquidity wallet, but the naming and usage suggest the contract is supposed to manage a separate liquidity resource or liquidity wallet rather than the team treasury.

### Impact

- The project may accidentally lock and release funds to the same address rather than the intended liquidity wallet.
- Ambiguity between `owner()`, `liquidityWallet`, and `msg.sender` creates governance confusion.
- Future changes to the wallet structure can break the lock accounting.

### Recommendation

Use a dedicated lock owner or liquidity wallet in the mapping and in transfers. For example:

```solidity
liquidityLocks[liquidityWallet] = LiquidityLock({ ... });
_transfer(liquidityWallet, address(this), amount);
_transfer(address(this), liquidityWallet, releasable);
```

This makes the lock semantics clearer and aligns with the intended liquidity-management design.

## 3) Medium - Anti-bot enforcement is not fully robust when pair configuration is incomplete

### Details

The anti-bot logic is enforced only when `launchTimestamp` is set and the current block is still within the anti-bot window:

```solidity
if (launchTimestamp == 0 || block.timestamp > launchTimestamp + antiBotWindow) {
    return;
}
```

Then the checks operate on `pairAddress` and wallet limits:

```solidity
if (!limitExempt[to] && to != pairAddress && to != DEAD) {
    require(balanceOf(to) + amount <= maxWalletAmount, "Wallet limit exceeded");
}

if (from == pairAddress) {
    require(amount <= maxBuyAmount, "Buy limit exceeded");
}

if (to == pairAddress) {
    require(amount <= maxSellAmount, "Sell limit exceeded");
}
```

This assumes the pair address is set before launch and remains correct. If the pair is not set, the contract can fail to enforce the intended trade limits or allow weird transfer patterns around the launch window.

### Impact

- The launch protection is only as good as the pair configuration.
- A missing or invalid pair decreases the reliability of the anti-bot hardening.
- Misconfigured early trading can expose the project to bot activity or harmful front-running.

### Recommendation

- Require `pairAddress` to be set before enabling trading.
- Validate the pair address in `enableTrading()`.
- Consider locking the anti-bot window until the pair is configured and liquidity is established.

## 4) Low - Tax distribution logic requires pairing with correct wallet setup

### Details

The tax is split as follows:

```solidity
marketingTaxShare = 50;
liquidityTaxShare = 30;
burnTaxShare = 20;
```

and enforced in `_transferWithTax`:

```solidity
uint256 marketingAmount = (taxAmount * marketingTaxShare) / 100;
uint256 liquidityAmount = (taxAmount * liquidityTaxShare) / 100;
uint256 burnAmount = taxAmount - marketingAmount - liquidityAmount;
```

This is mathematically consistent, but it depends heavily on `marketingWallet`, `liquidityWallet`, and `pairAddress` being configured correctly. If the project changes wallets or routes without updating tax exemptions, the token can produce unexpected behavior.

### Recommendation

Add wallet validation and state checks whenever these addresses change so that tax recipients and liquidity destinations remain consistent with launch conditions.

## Additional Observations

- The trade gate `if (!tradingEnabled && !taxExempt[from] && !taxExempt[to]) revert("Trading not enabled");` is reasonable and prevents trading before launch.
- The `nonReentrant` guard is present on transfer paths, which is good practice.
- The use of a blacklist and tax exemptions is straightforward and consistent with token launch controls.
- The code is structured for a launch token and is not obviously malicious by itself, but the vesting/misrouting issues should be corrected before deployment or fund distribution.

## Risk Rating

- Security posture: Moderate
- Main concern: Incorrect vesting release flow
- Recommended action: Fix vesting and liquidity routing before launch; then re-run a second-pass review on wallet configuration and tax enforcement.

## Confirmed Conclusion

The contract is not production-safe as written because the vesting release function transfers released tokens to the owner instead of the beneficiary. This is a clear logic bug and should be treated as a high-priority remediation item.
