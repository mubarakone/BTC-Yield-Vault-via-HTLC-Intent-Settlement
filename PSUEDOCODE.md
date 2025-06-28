Below is a **proposed architecture flow** plus a **Solidity-style EVM contract sketch** showing how you could wire up:

1. a **native‐BTC HTLC** on Bitcoin,
2. an **off-chain intent + solver**, and
3. an **on-chain “Yield Vault”** that holds and pays out yield upon preimage reveal.

---

## 1. Sequence Diagram (text form)

```plaintext
User                               Solver                             EVM Vault Contract                       Bitcoin Blockchain
│                                     │                                      │                                      │
│ 1) Pick secret preimage S          │                                      │                                      │
│    compute H = sha256(S)           │                                      │                                      │
│─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────▶│
│                                     │                                      │  (HTLC script: can be claimed with S ──▶ lock BTC  
│                                     │                                      │   or refunded by user after timelock)   │
│                                     │                                      │                                      │
│ 2) User funds HTLC with H & timelock│                                      │                                      │
│─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────▶│
│                                     │                                      │                                      │
│ 3) User calls registerIntent(H,     │                                      │                                      │
│    userEvmAddr, yieldAmount, expiry)│───────────────────────────────────▶│ registerIntent(…)                     │
│                                     │                                      │    stores intent{H, user, yieldAmt,    │
│                                     │                                      │    expiry, funded=false, done=false}   │
│                                     │                                      │                                      │
│                                     │ 4) Solver sees HTLC on Bitcoin      │                                      │
│                                     │    and intent in EVM vault          │                                      │
│                                     │──────────────────────────────────────────────────────────────────────────────────────────▶│
│                                     │                                      │                                      │
│                                     │ 5) Solver fundYield(H)              │                                      │
│                                     │───────────────────────────────────▶│ fundYield(H)                           │
│                                     │                                      │    require before expiry,             │
│                                     │                                      │    pull yieldAmount from solver;      │
│                                     │                                      │    set funded=true                     │
│                                     │                                      │                                      │
│                                     │                                      │                                      │
│                                     │ 6) Solver claims BTC from HTLC by   │                                      │
│                                     │    revealing S on Bitcoin           │                                      │
│                                     │──────────────────────────────────────────────────────────────────────────────────────────▶│
│                                     │                                      │                                      │
│                                     │ 7) Solver notifies EVM vault with S │                                      │
│                                     │───────────────────────────────────▶│ revealPreimage(S)                      │
│                                     │                                      │    require sha256(S)==H;              │
│                                     │                                      │    set done=true;                     │
│                                     │                                      │    emit PreimageRevealed(H, S);       │
│                                     │                                      │                                      │
│                                     │                                      │                                      │
│ 8) Vault auto‐pays out or user calls│                                      │                                      │
│    claimYield(H, S)                 │◀───────────────────────────────────│ on revealPreimage or on claimYield:   │
│                                     │                                      │    transfer yieldAmount to user       │
│                                     │                                      │                                      │
│                                     │                                      │                                      │
│ 9) If expiry passes without preimage│                                      │                                      │
│    solver can call refundYield(H)   │◀───────────────────────────────────│ refundYield: return yield to solver   │
```

---

## 2. EVM-Side “YieldVault” Contract Sketch

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IERC20 { 
    function transferFrom(address from, address to, uint amt) external returns(bool);
    function transfer(address to, uint amt) external returns(bool);
}

contract BTCYieldVault {
    struct Intent {
        address user;
        address solver;
        uint256 yieldAmount;
        uint256 expiry;         // block.timestamp after which refund is allowed
        bool funded;
        bool done;
    }

    IERC20 public immutable yieldToken; // e.g. USDC
    mapping(bytes32 => Intent) public intents;

    event IntentRegistered(bytes32 indexed H, address user, uint256 yieldAmount, uint256 expiry);
    event YieldFunded(bytes32 indexed H, address solver);
    event PreimageRevealed(bytes32 indexed H, bytes32 preimage);
    event YieldClaimed(bytes32 indexed H, address user);
    event YieldRefunded(bytes32 indexed H, address solver);

    constructor(address _yieldToken) {
        yieldToken = IERC20(_yieldToken);
    }

    /// 1) Borrower registers their intent after funding the BTC‐HTLC
    function registerIntent(
        bytes32 H,
        address userEvm,
        uint256 yieldAmt,
        uint256 timelockExpiry
    ) external {
        require(intents[H].user == address(0), "Already registered");
        require(block.timestamp < timelockExpiry, "Already expired");
        intents[H] = Intent({
            user: userEvm,
            solver: address(0),
            yieldAmount: yieldAmt,
            expiry: timelockExpiry,
            funded: false,
            done: false
        });
        emit IntentRegistered(H, userEvm, yieldAmt, timelockExpiry);
    }

    /// 2) Solver deposits yield tokens up front
    function fundYield(bytes32 H) external {
        Intent storage it = intents[H];
        require(it.user != address(0), "No such intent");
        require(!it.funded, "Already funded");
        require(block.timestamp < it.expiry, "Intent expired");
        // pull in yield tokens from solver
        require(yieldToken.transferFrom(msg.sender, address(this), it.yieldAmount),
                "Transfer failed");
        it.solver = msg.sender;
        it.funded = true;
        emit YieldFunded(H, msg.sender);
    }

    /// 3) Solver (or anyone) reveals the BTC‐HTLC preimage
    function revealPreimage(bytes32 S) external {
        bytes32 H = keccak256(abi.encodePacked(S));
        Intent storage it = intents[H];
        require(it.funded, "Not funded");
        require(!it.done, "Already done");
        require(block.timestamp < it.expiry, "Expired");
        it.done = true;
        // send yield to user
        require(yieldToken.transfer(it.user, it.yieldAmount), "Payout failed");
        emit PreimageRevealed(H, S);
        emit YieldClaimed(H, it.user);
    }

    /// 4) If solver never fulfills, they can reclaim their tokens after expiry
    function refundYield(bytes32 H) external {
        Intent storage it = intents[H];
        require(it.funded, "Not funded");
        require(!it.done, "Already done");
        require(block.timestamp >= it.expiry, "Not yet expired");
        require(msg.sender == it.solver, "Only solver");
        it.done = true;
        require(yieldToken.transfer(it.solver, it.yieldAmount), "Refund failed");
        emit YieldRefunded(H, it.solver);
    }
}
```

### How It Fits Together

1. **BTC-HTLC (on Bitcoin)**

   * Script locks BTC under hash `H` and timelock `T`.
   * Either the solver claims with preimage `S` before `T`, or user reclaims after `T`.

2. **Intent Registration (on EVM)**

   * User calls `registerIntent(H, userAddr, yieldAmt, expiryBlock)` after funding HTLC.

3. **Solver Funds Yield**

   * Solver locks up `yieldAmt` of an ERC-20 (e.g. USDC) by calling `fundYield(H)`.

4. **Swap/Deployment of Capital**

   * Off-chain, solver deploys capital to DeFi (Aave, Curve, etc.) to generate real yield.

5. **Preimage Reveal & Claim**

   * Once solver has claimed BTC on Bitcoin (revealing `S`), they submit `revealPreimage(S)` on EVM.
   * Vault immediately pays out the ERC-20 yield to the user.

6. **Fail-Safe**

   * If solver never acts by `expiry`, they can call `refundYield(H)` to get back their tokens.

---

### Key Properties

* **BTC never leaves Bitcoin** — locked only in a HTLC script.
* **Yield paid in ERC-20** on EVM side, enforceable by smart contracts.
* **Trust-minimized** — solver must fund tokens up-front or lose them.
* **Refund paths** exist on both chains if either party fails to perform.

—

Let me know if you’d like to drill into the exact Bitcoin‐script for the HTLC, or if you want to see how slashing/liquidation agents might be integrated!
