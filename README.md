# Smart Contract Visual Architecture & Factory Models

Interactive, step-by-step graphical models and architectural teardowns of Ethereum smart contracts, illustrating execution pipelines, storage layouts, access controls, and state transitions.

---

## Featured: Tether USD (USDT) Contract Anatomy

**Ethereum Mainnet Contract:** [`0xdAC17F958D2ee523a2206206994597C13D831ec7`](https://etherscan.io/address/0xdAC17F958D2ee523a2206206994597C13D831ec7#code)  
**Source:** `pragma solidity ^0.4.17`, compiled with `v0.4.18+commit.9cf6e910`  
**Inheritance:** `TetherToken is Pausable, StandardToken, BlackList`  
**Decimals:** 6 (1 USDT = 1,000,000 base units)

The USDT contract is structured as a pipeline where state mutations occur sequentially and deterministically. Any failed check throws, which reverts the whole call.

---

### The 5-Stage `transfer` Pipeline

```
[ 1. Input Dock ] ──▶ [ 2. Checkpoint Scanner ] ──▶ [ 3. Toll Booth ] ──▶ [ 4. Master Ledger ] ──▶ [ 5. Delivery Dock ]
  (msg.sender,          (!paused,                     (payload size;        (SafeMath sub/add;      (emit Transfer;
   _to, _value)          !isBlackListed[sender])       fee = value*bps/      fee credited to         no return value)
                                                       10000, capped; 0)     owner)
                              │                                                   │
                              ├──▶ (deprecated) Forward to upgradedAddress        └──▶ (balance too low) SafeMath throws
                              ▼ (check fails)
                      [ Revert ]
```

1. **Stage 1: Input Dock (Ingestion)**  
   Receives `transfer(_to, _value)`, `transferFrom(_from, _to, _value)` or `approve(_spender, _value)` along with `msg.sender`.
2. **Stage 2: Checkpoint Scanner (Guards)**  
   - `whenNotPaused` → `require(!paused)`: blocks transfers while the owner has paused the contract. `approve` is not pause-gated.
   - `require(!isBlackListed[msg.sender])` in `transfer`, `require(!isBlackListed[_from])` in `transferFrom`. **Only the sender is checked**; a blacklisted address can still receive tokens.
   - **Deprecation is a fork, not a guard:** if `deprecated` is true, the call is forwarded to `UpgradedStandardToken(upgradedAddress).transferByLegacy(...)` (or `transferFromByLegacy` / `approveByLegacy`) and the legacy ledger is never touched.
3. **Stage 3: Toll Booth (Fee Deduction)**  
   First, `BasicToken.transfer`'s `onlyPayloadSize(2 * 32)` rejects calldata shorter than expected (short-address attack fix). Then:
   $$\text{fee} = \min\left(\left\lfloor\frac{\text{value} \times \text{basisPointsRate}}{10000}\right\rfloor, \text{maximumFee}\right)$$
   Both `basisPointsRate` and `maximumFee` are currently 0. `setParams` hardcodes ceilings: `newBasisPoints < 20` (under 0.2%) and `newMaxFee < 50` whole tokens.
4. **Stage 4: Master Ledger (Storage Mutation)**  
   There is no explicit balance `require`; SafeMath's `sub` throws on underflow instead.
   ```solidity
   uint sendAmount = _value.sub(fee);
   balances[msg.sender] = balances[msg.sender].sub(_value);   // throws if insufficient
   balances[_to] = balances[_to].add(sendAmount);
   if (fee > 0) {
       balances[owner] = balances[owner].add(fee);
       Transfer(msg.sender, owner, fee);
   }
   ```
   `transferFrom` also spends `allowed[_from][msg.sender]`, unless the allowance is `MAX_UINT` (infinite approval).
5. **Stage 5: Delivery Dock (Receipt Dispatch)**  
   Emits `Transfer(from, to, sendAmount)`. `transfer`, `transferFrom` and `approve` **return nothing**, which is why integrators must use a "safe transfer" wrapper rather than expecting the ERC-20 `bool`.

---

### Privileged & Administrative Powers (`onlyOwner`)

- **Supply Vault (`_totalSupply`):** No cap. `issue(amount)` mints into the owner's balance; `redeem(amount)` burns from the owner's balance (requires the owner to hold enough). Emits `Issue` / `Redeem`.
- **Ownership (`transferOwnership`):** Single-step. Ownership moves immediately to the new address (a zero address is silently ignored). There is no `claimOwnership` handshake.
- **Blacklist (`addBlackList` / `removeBlackList`):** Freezes outgoing transfers from an address.
- **Blacklist Confiscation (`destroyBlackFunds`):** Only for already blacklisted addresses. Sets their balance to 0 and decreases `_totalSupply`. Emits `DestroyedBlackFunds`.
- **Emergency Stop (`pause` / `unpause`):** Halts `transfer` and `transferFrom` for everyone.
- **Upgrade (`deprecate(upgradedAddress)`):** One-way. Forwards all ERC-20 calls and reads (`balanceOf`, `allowance`, `totalSupply`) to a new contract.
- **Fees (`setParams`):** Turns on the transfer fee within the hardcoded ceilings above. Fees go to the owner.

---

## Interactive Visual Simulator

The repository includes a standalone interactive HTML/SVG visualization, **The USDT Bureau** ([`index.html`](index.html)). Each pipeline stage is a clerk at a desk, and the transfer request is a paper form carried from desk to desk:

| Clerk | Stage | What they do |
|---|---|---|
| 📋 Ada, Intake | 1. Input Dock | Reads the request and converts the amount to 6-decimal base units |
| 🔍 Brandt, Inspector | 2. Checkpoint | `whenNotPaused`, sender blacklist, deprecation (sends the form to the New Office) |
| 🧮 Pell, Fee Collector | 3. Toll Booth | `onlyPayloadSize`, fee calculation and the `maximumFee` cap |
| 📖 Mora, Bookkeeper | 4. Master Ledger | SafeMath subtract and add in `balances`, fee to the owner (the "Director") |
| 📯 Tobin, Town Crier | 5. Delivery Dock | Emits `Transfer`; returns nothing |

Rejected forms go to **the Shredder** (revert). The simulator features:
- Slow-motion playback, one check per step, with a speed setting, Pause and single-Step.
- Each clerk explains their check in plain words, with the matching Solidity line highlighted alongside.
- The Director's orders: pause, deprecate, `setParams` within its ceilings, `issue`, `redeem`, blacklist and `destroyBlackFunds`.
- A live balance book and a public notice board of events.

The checks run in the same order as the code. `onlyPayloadSize` sits on `BasicToken.transfer`, so it runs after the pause, blacklist and deprecation checks, at the start of stage 3.

It is a model for learning, not a connection to the chain; no wallet or network access is involved.

### Running Locally

Open `index.html` in any modern web browser or serve locally:

```bash
python3 -m http.server 8080
```
Then navigate to `http://localhost:8080`.
