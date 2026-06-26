# Builder's Guide: The Dynamic Attestation Fee

## 1. Purpose

The Binary Journal protocol creates the world's first truth-backed economy. Every time a user attests a truth, they pay a fee in PLS that is permanently locked into a liquidity pool for the BIJO token. The fee must:

- Be high enough to build a meaningful liquidity floor after a modest number of truths.
- Stay accessible to individuals recording personal or family history.
- Adjust automatically so that the dollar cost remains constant regardless of PLS price volatility.

This guide explains the fee design, the oracle, and the smart-contract implementation.

---

## 2. Fee Structure

| Truth type | Target USD cost | Purpose |
|---|---|---|
| Light (public) | $20.00 | Each Light truth injects significant value into the commons. |
| Dark (private/heir) | $0.0008 | Keeps family archiving essentially free while still contributing a minimum. |

At current PLS price (~$0.000008) these targets map to 2,500,000 PLS (Light) and 100 PLS (Dark).

The target USD values are hard-coded as constants in the LiquidityReserve contract. They can be changed by the owner only during the Verification Period, then frozen before the ownership burn.

---

## 3. Dynamic PLS Calculation

Because PLS price fluctuates, the contract does not store a fixed PLS fee. Instead it stores:

- `plsPriceInUsd` – the current price of 1 PLS in USD, expressed with 18 decimals (e.g. 0.000008 USD → 8000000000000).

This price is updated periodically by the contract owner (or later by a trustless oracle).

The required PLS for an attestation is calculated on-chain at the moment of the transaction:

```
requiredPLS = (TARGET_USD * 10^18) / plsPriceInUsd
```

**Example (Light, $20 target, PLS = $0.000008):**

```
requiredPLS = (20 * 10^18) / 8000000000000 = 2,500,000 PLS
```

**Example after PLS price doubles to $0.000016:**

```
requiredPLS = (20 * 10^18) / 16000000000000 = 1,250,000 PLS
```

The user always pays roughly $20 worth of PLS, no matter the market.

---

## 4. Oracle Design (Owner-Updated → Immutable)

During the Verification Period, the contract owner (the founder) updates `plsPriceInUsd` by calling `setPrice(uint256 _price)`. This is a trust-based step, but it is temporary. The value used can be sourced from:

- PulseX TWAP of PLS/DAI or PLS/USDC.
- Manual feed from a trusted price API.

Before the Ownership Burn (end of Verification Period), the owner will either:

- **Option A**: Freeze the fee by removing the dynamic logic and hardcoding a fixed PLS fee that reflects a fair market price at that moment.
- **Option B**: If a reliable decentralized oracle for PLS/USD exists on PulseChain by then, replace the owner-setter with that oracle's price feed and then renounce ownership.

After the burn, the mechanism becomes immutable.

---

## 5. Contract Implementation (LiquidityReserve.sol)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "@openzeppelin/contracts/access/Ownable.sol";

contract LiquidityReserve is Ownable {
    // Target costs in USD, 18 decimal places
    uint256 public constant LIGHT_TARGET_USD = 20 * 10**18;   // $20.00
    uint256 public constant DARK_TARGET_USD  = 8 * 10**14;    // $0.0008

    // Price of 1 PLS in USD, 18 decimals (e.g. 0.000008 => 8000000000000)
    uint256 public plsPriceInUsd;

    // Total PLS collected (for informational purposes)
    uint256 public totalPlsCollected;

    event PriceUpdated(uint256 newPrice);
    event AttestationPaid(address indexed user, uint256 amount, bool isLight);

    constructor() Ownable(msg.sender) {
        // Initial price will be set immediately after deployment
    }

    function setPrice(uint256 _price) external onlyOwner {
        plsPriceInUsd = _price;
        emit PriceUpdated(_price);
    }

    // Light truth attestation – call with PLS attached
    function attestLight() external payable {
        uint256 required = (LIGHT_TARGET_USD * 10**18) / plsPriceInUsd;
        require(msg.value >= required, "Insufficient PLS for light truth");
        totalPlsCollected += required;
        // Refund any excess
        if (msg.value > required) {
            payable(msg.sender).transfer(msg.value - required);
        }
        emit AttestationPaid(msg.sender, required, true);
    }

    // Dark truth attestation
    function attestDark() external payable {
        uint256 required = (DARK_TARGET_USD * 10**18) / plsPriceInUsd;
        require(msg.value >= required, "Insufficient PLS for dark truth");
        totalPlsCollected += required;
        if (msg.value > required) {
            payable(msg.sender).transfer(msg.value - required);
        }
        emit AttestationPaid(msg.sender, required, false);
    }

    // Allow the contract to receive PLS directly (fallback)
    receive() external payable {
        totalPlsCollected += msg.value;
    }
}
```

**Note:** The `attestLight` and `attestDark` functions are separate for clarity, but the actual attestation (calling `Attestation.attest()`) is done in the same transaction by the app, not inside this contract. The app will call `LiquidityReserve.attestLight{value: requiredPLS}()` first, then call `Attestation.attest(hash)`. This separation keeps the reserve contract simple and the attestation logic in its own immutable contract.

---

## 6. App Integration Flow (Frontend / Mobile)

1. User selects "Attest" for a truth in the bijo-app.
2. App reads `plsPriceInUsd` from LiquidityReserve and calculates the required PLS for the truth type (Light/Dark).
3. App prompts user to confirm the fee, showing the dollar equivalent (e.g. "This will cost ~$20.00 in PLS").
4. User confirms; app executes two transactions:
   - `LiquidityReserve.attestLight{value: required}()` or `attestDark{value: required}()`.
   - `Attestation.attest(hash)`.
5. Optionally, app also calls `StorageEndowment.allocate(hash)` to lock the BIJO endowment.

All three steps (fee, attestation, endowment) can be batched into a single meta-transaction or relayed by the app's burner wallet.

---

## 7. Liquidity Pool Deployment (After Verification)

During verification, the LiquidityReserve simply accumulates PLS. When the protocol moves to Phase 3 (Liquidity & Transfer Enablement):

1. The owner calls `BIJO.enableTransfers()`.
2. The owner (or a permissionless script) deploys a PulseX liquidity pool:
   - Withdraws all PLS from LiquidityReserve (requires a one-time withdrawal function – can be added with `onlyOwner`).
   - Pairs it with an equal value of BIJO from the Ecosystem Reserve (the 13.5% multisig).
   - Adds liquidity to PulseX and burns the LP tokens.
3. After this step, the LiquidityReserve can be abandoned or repurposed; its PLS has now become permanent, unruggable liquidity.

---

## 8. Effect on BIJO Market

Because every Light truth adds $20 of permanent buy-side liquidity, the pool's size grows with adoption. The table below shows the pool value for different numbers of Light truths, assuming Dark truths contribute negligible amounts:

| Light Truths | Total PLS Reserve | Pool Value (paired with BIJO) |
|---|---|---|
| 100 | 250,000,000 PLS | ~$2,000 |
| 500 | 1,250,000,000 PLS | ~$10,000 |
| 1,000 | 2,500,000,000 PLS | ~$20,000 |
| 10,000 | 25,000,000,000 PLS | ~$200,000 |

After 1,000 Light truths, BIJO has a $20,000 permanent floor – enough to give it real market value and make node operators' earnings meaningful.

---

## 9. Future Trustless Oracle

Once the protocol is immutable, the price feed must either be frozen or decentralized. Options for a future upgrade (which would require deploying a new LiquidityReserve and migrating the PLS) include:

- Integrating a Chainlink PLS/USD feed (if it becomes available on PulseChain).
- Using a PulseX TWAP oracle that can be queried trustlessly on-chain.
- A community-governed medianizer of multiple trusted price feeds.

Until then, the temporary owner-updated mechanism is a practical bridge, and the community can monitor its accuracy via the emitted `PriceUpdated` events.

---

## 10. Summary for Builders

- The attestation fee is not a fixed PLS amount. It's a constant dollar value translated to PLS via an on-chain price variable.
- Light truths target **$20**, Dark truths target **$0.0008**.
- The LiquidityReserve contract holds the price, exposes `setPrice()` for the owner, and provides payable `attestLight()`/`attestDark()` functions.
- The app calculates the required PLS, asks for user confirmation, and sends the fee.
- All PLS collected becomes permanent BIJO liquidity after the verification period.
- Before the ownership burn, the price mechanism is either frozen or migrated to a trustless oracle.

---

*This design ensures that the truth-based economy is not just a metaphor – it's a mathematically precise engine that converts human witness into unstoppable market depth.*
