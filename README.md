# Smart Contract Visual Architecture & Factory Models

Interactive, step-by-step graphical models and architectural teardowns of Ethereum smart contracts, illustrating execution pipelines, storage layouts, access controls, and state transitions.

---

## Featured: Tether USD (USDT) Contract Anatomy

**Ethereum Mainnet Contract:** [`0xdAC17F958D2ee523a2206206994597C13D831ec7`](https://etherscan.io/address/0xdAC17F958D2ee523a2206206994597C13D831ec7)  
**Compiler:** Solidity 0.4.18  
**Decimals:** 6 (1 USDT = 1,000,000 base units)

The USDT contract is structured as an event-driven industrial pipeline where state mutations occur sequentially and deterministically.

---

### The 5-Stage Execution Pipeline

```
[ 1. Input Dock ] ──▶ [ 2. Checkpoint Scanner ] ──▶ [ 3. Toll Booth ] ──▶ [ 4. Master Ledger ] ──▶ [ 5. Delivery Dock ]
  (msg.sender,          (!deprecated,                  (basisPointsRate       (balances[from] -= v    (emit Transfer,
   to, value)            !blackList,                    fee cut, 0%)           balances[to] += v)      return true)
                         balance >= v)
                              │
                              ▼ (on failure)
                      [ Blacklist Furnace / Revert ]
                        (destroyBlacklisted)
```

1. **Stage 1: Input Dock (Ingestion)**  
   Validates incoming transactions (`transfer`, `transferFrom`, `approve`) with signed calldata, caller address (`msg.sender`), and gas limits.
2. **Stage 2: Checkpoint Scanner (Guards)**  
   Enforces contract assertions:
   - `require(!deprecated)`: Asserts contract pause state is inactive.
   - `require(!blackList[msg.sender] && !blackList[to])`: Blocks frozen addresses.
   - `require(balances[msg.sender] >= value)`: Verifies solvent balance.
   - *Failure:* Execution immediately reverts.
3. **Stage 3: Toll Booth (Fee Deduction)**  
   Calculates protocol transfer fee:
   $$\text{fee} = \min\left(\frac{\text{value} \times \text{basisPointsRate}}{10000}, \text{maximumFee}\right)$$
   *(Configured to 0 bps on mainnet).*
4. **Stage 4: Master Ledger (Storage Mutation)**  
   Mutates internal EVM storage mappings:
   ```solidity
   balances[from] -= value;
   balances[to] += (value - fee);
   ```
5. **Stage 5: Delivery Dock & Receipt Dispatch**  
   Emits public event logs (`Transfer`, `Approval`) indexed by block explorers and returns `true`.

---

### Privileged & Administrative Powers

- **Supply Vault (`totalSupply`):** No algorithmic ceiling. The owner can call `issue(amount)` to mint new tokens directly into circulation or `burn(amount)` to retire supply.
- **Two-Step Ownership Handover:** Prevents accidental loss of contract authority via a two-stage handshake (`transferOwnership(newOwner)` followed by `claimOwnership()`).
- **Blacklist Confiscation (`destroyBlacklisted`):** Permanently erases token balances belonging to sanctioned or frozen addresses and decreases global `totalSupply`.
- **Soft Pause (`deprecate`):** Halts transfers without locking balances or blocking administrative recovery.

---

## Interactive Visual Simulator

The repository includes a standalone interactive HTML/SVG visualization (`index.html`) demonstrating:
- Step-by-step transaction tracing across the assembly pipeline.
- Interactive toggle controls for emergency pause barriers and blacklist trapdoors.
- Real-time simulation of transfers, reverts, and administrative mint events.

### Running Locally

Open `index.html` in any modern web browser or serve locally:

```bash
python3 -m http.server 8080
```
Then navigate to `http://localhost:8080`.
