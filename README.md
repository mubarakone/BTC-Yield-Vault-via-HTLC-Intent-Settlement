To allow someone to **earn yield on their native BTC** — without wrapping it — using a **Garden-inspired HTLC + intent-based model**, you'd flip the lending flow and design a **non-custodial BTC-to-EVM yield pipeline**.

---

## ✅ Goal: Let BTC holders earn yield **without wrapping or giving up custody**

This means:

* BTC stays on the Bitcoin blockchain (unwrapped).
* Yield is earned in the form of **ETH, USDC, or another EVM token**.
* The system uses **HTLCs + solver coordination** to match BTC lenders with DeFi opportunities on EVM chains (e.g., Aave, Curve, Pendle, etc.).

---

## 🧠 Core Model: **BTC Yield Vault via HTLC + Intent Settlement**

### Step-by-Step Flow:

#### 🔐 1. **User time-locks BTC in an HTLC**

* User locks BTC in an HTLC (with a 30–90 day duration).
* The contract:

  * Has a hashlock (secret only known to the protocol),
  * Has a refund path (after timelock expires),
  * Signals **intent to lend BTC**.

#### 📩 2. **User submits intent: “Lend BTC for yield”**

* This intent includes:

  * Amount of BTC,
  * Target chain and wallet (for yield payouts),
  * Minimum yield requirement (e.g., “at least 4% APY”),
  * Term (e.g., 30 days),
  * Associated HTLC hash.

#### 🤖 3. **Solver matches BTC with EVM DeFi opportunity**

* Solvers (or protocol relayers) verify the locked BTC.
* They deploy **protocol capital** to:

  * Use BTC as pseudo-collateral,
  * Mint USDC on Aave using protocol liquidity,
  * Deploy that USDC to generate yield (e.g., Pendle, Yearn),
  * Reserve yield for the BTC lender.

#### 🪙 4. **Yield is delivered to BTC lender**

* At the end of the HTLC period:

  * BTC holder **claims their BTC back** using the refund path,
  * Or the HTLC is closed early by the solver, who reveals the preimage if the yield was paid.

* BTC holder receives **ETH, USDC, or stablecoin yield** to their EVM address.

  * Can be **streamed**, **paid upfront**, or **settled at the end** depending on design.

---

## ✅ Result:

* User’s BTC never leaves the Bitcoin chain.
* User earns **on-chain yield** (paid in EVM tokens).
* No wrapping. No custodians. No centralized platform.

---

## 💡 Alternative Architecture: BTC-Powered Yield Pools

| Model                  | Description                                                                                    |
| ---------------------- | ---------------------------------------------------------------------------------------------- |
| **Shared BTC Vault**   | Multiple users lock BTC into HTLCs; solvers batch them into DeFi strategies.                   |
| **Yield Tokenization** | Protocol could issue a temporary yield token (e.g., `yBTC`) on Ethereum to track yield rights. |
| **Prepaid Yield**      | Solvers pay the yield upfront, then earn back by deploying user BTC indirectly.                |

---

## ⚠️ Challenges

| Issue                        | Details                                                                 |
| ---------------------------- | ----------------------------------------------------------------------- |
| **No BTC-native yield**      | Yield must come from the EVM side; BTC is passive collateral.           |
| **Solver trust assumptions** | Yield disbursement depends on external actors.                          |
| **Rate transparency**        | Requires a pricing oracle or market for BTC yield opportunities.        |
| **Limited scripting**        | BTC can’t enforce “yield return” — must be trust-minimized using HTLCs. |

---

## ✅ Summary: How to Earn Yield on Native BTC

| Step                            | Action                                           |
| ------------------------------- | ------------------------------------------------ |
| 1. Lock BTC in HTLC             | Funds secured with hashlock + timelock           |
| 2. Submit off-chain intent      | Signals lending preferences and wallet for yield |
| 3. Solver matches yield         | Deploys capital on EVM; commits to deliver yield |
| 4. BTC is refunded + yield paid | BTC never leaves chain; yield sent to EVM wallet |

---

Would you like a specific architecture diagram, or a contract flow for how the preimage + yield disbursement might work in Solidity?
