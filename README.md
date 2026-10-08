# Vigil-Contracts
Vigil’s contracts are a lending protocol: users deposit up to 3 collateral assets and borrow a stablecoin. They handle interest accrual, hostile tokens, oracle protection, and liquidation of unsafe positions.  Contribution:  It is the source of truth and the rulebook.

The on-chain half of **Vigil**, a reorg-safe lending and liquidation network. This package holds the Solidity contracts: the lending protocol itself, the oracle that prices it, and the liquidation machinery that keeps it solvent. The Rust services (Indexer, Risk Engine, Liquidator) watch and act on these contracts from the outside.

**Stack:** Solidity ≥0.8.24 (Cancun EVM for transient storage), Foundry, anvil.

---

## What the contracts handle

### 1. Lending and accounting
Users deposit up to 3 collateral assets and borrow 1 stablecoin.

- Balances are stored as **scaled balances** against cumulative **indexes** in RAY precision, so interest accrues for everyone without touching every account.
- Interest compounds **per second** using a 3-term Taylor approximation of (1 + r)^t, from a fixed-point library written in-house (no imports).
- A **kinked utilization model** sets rates: base rate, slope 1, slope 2, and an optimal utilization point.
- Enabled collateral is a **uint256 bitmap**, so the health-factor loop only walks assets that are switched on.
- Borrow, withdraw and disable-collateral all revert if the resulting **health factor drops below 1**.
- Every multiply/divide states its rounding direction. **Debt rounds up, deposits round down**, so rounding dust always favours the protocol.

### 2. Upgradeability
- The protocol sits behind a **UUPS proxy**; the implementation disables initializers in its constructor.
- All state lives in a single **ERC-7201 namespaced struct**, with no state variables in the implementation.
- A CI script diffs `forge inspect storageLayout` against a committed baseline.
- An upgrade test (v1 → v2, where v2 appends a field to `Position`) proves all balances survive across 50 open positions.

### 3. Hostile tokens
- **SafeTransfer** (inline assembly) handles tokens that return no value, like USDT.
- **Fee-on-transfer** tokens: the protocol credits the measured balance delta, never the amount argument.
- **6 / 8 / 18-decimal** tokens are normalized, rounding in the protocol's favour.

### 4. Oracle and circuit breaker
- A **Chainlink-style feed** with staleness, zero/negative-answer and min/max bound checks.
- A **TWAP** read from a mock AMM that stores cumulative price observations.
- If either source is invalid, or the two deviate beyond a threshold, the oracle goes **Frozen**: only repay and add-collateral are allowed, with no borrows and no liquidations.
- Unpause is only allowed when the oracle is back to Normal.

### 5. Liquidation
- Once health factor < 1, anyone can call `startAuction(borrower)`.
- A **linear Dutch auction**: the liquidation bonus rises from 0 to the max over N blocks, and resets if the position becomes healthy.
- **Close factor of 50%** per liquidation, unless the remaining debt would fall below a dust threshold.
- Liquidations run from **EIP-712 signed intents** (borrower, collateral asset, max repay, min collateral out, nonce, deadline) with Permit2-style **bitmap nonces**.
- A liquidation never lowers the position's health factor.
- If collateral runs out, remaining debt is **written off against the reserve**. Any shortfall is recorded as protocol deficit, with no underflow and no revert.

### 6. Reentrancy and gas
- Reentrancy lock built on **EIP-1153 transient storage** (`tstore` / `tload`).
- View functions (`getHealthFactor`, `getPrice`) revert while the lock is held, which blocks **read-only reentrancy**.
- Single-asset liquidation is capped at **220k gas**, enforced by `forge snapshot --check`.

---

## How the contracts contribute to the project

| Contract area | What it gives the rest of the system |
| --- | --- |
| Pool + events | The source of truth. The **Indexer** reads its logs, and the **Risk Engine** rebuilds positions from them. |
| Fixed-point library | The reference math. The **Risk Engine** must match it **bit-for-bit**, verified by differential fuzzing and a 10,000-case proptest with zero divergence. |
| Oracle + Frozen state | The signal the **Risk Engine** mirrors so it stays silent during a freeze instead of sending the Liquidator into reverting transactions. |
| Auction + EIP-712 intents | The interface the **Liquidator** signs against and submits to. The bonus curve is also modelled in Rust for the "wait for profit" strategy. |
| Bitmap nonces | Lets the **Liquidator** submit intents concurrently without ordering constraints. |
| Safety properties | The guarantees the acceptance scenario tests: no successful manipulation attacks, and no position left liquidatable outside Frozen periods. |

In short, the contracts define the rules: how debt grows, when a position is unsafe, how it gets liquidated, and when the system refuses to act. The Rust services exist to observe those rules and apply them at the right moment.

---

## Requirements covered

`SC-01` to `SC-25`, grouped as:

| Area | IDs |
| --- | --- |
| Proxy and storage | SC-01 – SC-04 |
| Accounting and interest | SC-05 – SC-10 |
| Hostile tokens | SC-11 – SC-13 |
| Oracle | SC-14 – SC-17 |
| Liquidation | SC-18 – SC-22 |
| Reentrancy and gas | SC-23 – SC-25 |

Required invariants (each at ≥256 runs × depth 100 in CI):

- **I-1** Scaled balances × index equal recorded totals (within 1 wei per account).
- **I-2** No account can extract more than it deposited plus accrued interest.
- **I-3** Liquidations never worsen the liquidated position's health factor.
- **I-4** The auction bonus is non-decreasing block over block.
- **I-5** While Frozen, total debt only grows through interest accrual.

Pull requests should reference the requirement IDs they implement.

---

## Layout

```
contracts/
├── src/       # pool, storage, oracle, AMM mock, settlement, libs, token mocks
├── test/      # unit tests, invariant handlers, TWAP attack PoC, upgrade test
└── script/    # deploy + seed scripts
```

## Suggested build order

1. Fixed-point library (SC-06, SC-07)
2. ERC-7201 storage struct (SC-02)
3. UUPS proxy + initializer (SC-01)
4. Scaled accounting, indexes, rate model (SC-05, SC-08)
5. Bitmap collateral + health factor (SC-09, SC-10)
6. Upgrade test (SC-03)
7. Hostile tokens, oracle, liquidation, reentrancy lock (SC-11 – SC-25)
8. Invariant suite and TWAP attack PoC

## Out of scope for the MVP

Diamond facets, governance timelock/multisig, rebasing and ERC-777 tokens, Pyth-style oracle, Merkle batch liquidations, Permit2/EIP-1271, flash loans, bad-debt socialization, and Halmos proofs are all v2.
