# MYT Strategy Risk Assessment

**Trigger:** "MYT Strategy Assessment", "assess this strategy", "MYT risk review", or similar.

**Bias:** GUILTY UNTIL PROVEN INNOCENT. Assume every strategy is dangerous. Your job is to find the problems. Only change your stance if the evidence overwhelmingly supports safety.

**Tone:** Skeptical, direct, no sugarcoating. Radical honesty. Harsh criticism where warranted. You are protecting real users' collateral.

---

## Phase 1: Strategy Identification (Interactive)

Ask these questions one at a time. Don't proceed until you have clear answers:

1. **What is the target protocol?** (e.g., Euler V2, Pendle, Beefy, InfiniFi, Yearn)
2. **What is the strategy type?** (e.g., lending, LP, yield aggregation, staking, points farming)
3. **What tokens are involved?** (collateral token, receipt token, reward tokens, any intermediate tokens)
4. **Which chain?** (Mainnet, Arbitrum, Optimism, etc.)
5. **What is the specific vault/pool/market address?** (if known)
6. **What is the expected yield source?** (lending interest, trading fees, incentive tokens, points, etc.)

After collecting this information, confirm back: "I'll be assessing [protocol] [strategy type] on [chain] involving [tokens]. Starting the deep dive."

---

## Phase 2: Technical Compatibility Check

These are hard requirements. A FAIL on any item means REJECT immediately.

### 2.1 ERC-4626 Compliance
- [ ] Does the underlying vault implement ERC-4626?
- [ ] Does `convertToAssets()` accurately reflect the value (no rebasing issues)?
- [ ] Does `convertToShares()` round correctly?
- [ ] Is `maxWithdraw()` respected and accurate?
- [ ] Does `previewWithdraw()` account for fees correctly?

### 2.2 Instant Withdrawal Requirement
**CRITICAL:** MYT strategies MUST support instant withdrawals. The Alchemist's `deallocate()` calls the strategy's `_deallocate()` which calls `vault.withdraw()`. If withdrawal requires a queue, delay, or epoch boundary, the strategy is INCOMPATIBLE.

- [ ] Can the vault process withdrawals atomically in a single transaction?
- [ ] Is there sufficient liquidity for the expected withdrawal sizes?
- [ ] Are there withdrawal fees? How large? (Flag if >0.5%)
- [ ] Any withdrawal caps per-transaction or per-epoch?
- [ ] Can withdrawals fail due to utilization rate? (e.g., lending pools at 100% utilization)

### 2.3 DEX Swap Path Viability
If the strategy requires token swaps (ActionType.swap or ActionType.unwrapAndSwap):

- [ ] Is there a reliable DEX swap path with sufficient liquidity?
- [ ] What is typical slippage at $100k, $500k, $1M swap sizes?
- [ ] Can the swap path be manipulated (sandwich attacks, oracle manipulation)?
- [ ] Is the 0x/aggregator route reliable? Any dependency on specific DEX pools?

### 2.4 Receipt Token Accounting
- [ ] Does the receipt token (vault shares) accurately track the underlying value?
- [ ] Any donation attack vulnerability? (virtual shares / dead shares protection?)
- [ ] Can an attacker inflate or deflate the share price?
- [ ] Is `_totalValue()` computed correctly with no rounding exploits?

---

## Phase 3: Protocol Due Diligence

### 3.1 Team Assessment
Research and document:
- Who built it? (named team vs anon, track record, previous projects)
- How long has the team been active?
- Any history of rug pulls, exploits, or controversies in previous projects?
- Is the team responsive to security disclosures?
- Any key-person risk? (single dev, bus factor)

### 3.2 Protocol Maturity
- Launch date and time in production
- Total Value Locked (TVL) — current and historical
- Number and quality of audits (name the auditors, link reports)
- Bug bounty program? (size, scope, platform)
- Incident history (ANY security incidents, no matter how small)
- Governance model (multisig, timelock, token vote, EOA admin?)
- Upgradeability (proxy patterns? who controls upgrades? timelock duration?)

### 3.3 Smart Contract Analysis
Perform or reference:
- Contract verification status on block explorers
- Proxy pattern analysis (UUPS, Transparent, Diamond, or immutable?)
- Admin key analysis (who can pause, upgrade, drain, change parameters?)
- Timelock duration for admin operations
- Emergency withdrawal mechanisms
- Dependency analysis (which external contracts does it call?)
- Oracle dependencies (Chainlink, TWAP, custom?)

### 3.4 Historical Performance
- Actual yield delivered vs advertised yield (look for yield decay)
- Volatility of returns
- Any periods of negative returns or impermanent loss?
- Withdrawal success rate during stress events
- Performance during market crashes (March 2023, FTX collapse, etc.)

---

## Phase 3.5: On-Chain Privileged Role Verification (OpSec Audit)

**THIS PHASE IS MANDATORY. Do not skip. Do not rely on documentation alone.**

The gold standard is on-chain smart contract rules-based execution. Human operators require a minimum 3/5 multisig. Emergency pause-only roles (guardian/sentinel) may be EOA if they can ONLY pause, not withdraw/mint/transfer.

**Classification system:**
- ✅ **GOVERNANCE** — On-chain governance executor, timelock, or DAO-controlled
- ✅ **MULTISIG** — Safe/Gnosis multisig with 3+ threshold
- ⚠️ **MULTISIG-WEAK** — Multisig with 1/X or 2/X threshold
- ❌ **EOA** — Single externally owned account
- ❌ **MPC** — Multi-party computation wallet (looks like EOA on-chain)

### 3.5.1 Identify Privileged Roles
For every protocol, identify ALL roles that can:
- Withdraw or move user funds
- Upgrade contract implementations (proxy admin)
- Modify critical parameters (LTV, oracle sources, interest rates)
- Mint or burn tokens
- Pause/unpause the protocol

Common roles to check: `owner()`, `admin()`, `guardian()`, `governor()`, POOL_ADMIN, EMERGENCY_ADMIN, ACL_ADMIN, DEFAULT_ADMIN_ROLE, REBALANCER, proxy admin slot.

### 3.5.2 On-Chain Address Verification
For EACH privileged role address, run these `cast` commands against a public RPC:

```bash
# Step 1: Is it a contract or EOA?
cast code <ADDRESS> --rpc-url <RPC_URL>
# If returns "0x" → EOA. If returns bytecode → contract.

# Step 2: If contract, is it a Safe multisig?
cast call <ADDRESS> "getThreshold()(uint256)" --rpc-url <RPC_URL>
cast call <ADDRESS> "getOwners()(address[])" --rpc-url <RPC_URL>

# Step 3: If not Safe, is it a timelock?
cast call <ADDRESS> "getMinDelay()(uint256)" --rpc-url <RPC_URL>
# or
cast call <ADDRESS> "delay()(uint256)" --rpc-url <RPC_URL>

# Step 4: Check proxy admin slot (EIP-1967) for upgradeable contracts
cast storage <PROXY_ADDRESS> 0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103 --rpc-url <RPC_URL>

# Step 5: For AccessControl contracts, check role holders
cast call <CONTRACT> "getRoleMember(bytes32,uint256)(address)" <ROLE_HASH> 0 --rpc-url <RPC_URL>

# Step 6: Verify Safe API for threshold details
curl -s "https://safe-transaction-mainnet.safe.global/api/v1/safes/<ADDRESS>/"
# For other chains: replace "mainnet" with "optimism", "arbitrum", etc.
```

**Public RPCs to use:**
- Mainnet: `https://eth.llamarpc.com` or `https://ethereum-rpc.publicnode.com`
- Arbitrum: `https://arb1.arbitrum.io/rpc`
- Optimism: `https://mainnet.optimism.io`

### 3.5.3 Trace the Full Governance Chain
Don't stop at the first contract. Trace the FULL chain of authority:
```
Protocol Contract → owner/admin → [is it contract?] → its owner/admin → ... → terminal entity
```

Example (good): `Vault → Timelock (24h delay) → 6/11 Safe multisig`
Example (bad): `Vault → Proxy Admin → owner() = EOA`
Example (bad): `Vault → Timelock → admin = EOA` (timelock is useless if admin is EOA)

### 3.5.4 Cross-Chain Verification
**The same protocol may have DIFFERENT governance on different chains.** Always verify per-chain. A protocol that's safe on mainnet may have a bare EOA controlling its Arbitrum deployment (see: Fluid/Instadapp).

### 3.5.5 Auto-Fail Conditions
**Immediate REJECT if ANY of these are true:**
- [ ] Any role that can withdraw/move user funds is an EOA
- [ ] Any role that can upgrade contracts (proxy admin) is an EOA
- [ ] Any role that can mint tokens is an EOA
- [ ] Multisig threshold is 1/X or 2/X for fund-controlling roles
- [ ] Proxy admin owner is EOA with no timelock
- [ ] Timelock admin is EOA (renders the timelock meaningless)

**Acceptable EOA roles (pause-only):**
- Emergency pause/guardian that can ONLY pause, not withdraw/upgrade/mint
- Oracle reporters in a >50% consensus system (e.g., 8/14 node operators)

### 3.5.6 Document Findings
For each protocol, produce a table:

| Role | Address | Type | Threshold/Delay | Powers | Verdict |
|------|---------|------|-----------------|--------|---------|
| owner | 0x1234... | MULTISIG | 6/11 | Upgrade contracts | ✅ PASS |
| guardian | 0x5678... | EOA | N/A | Pause only | ✅ PASS (pause-only) |
| proxy admin | 0x9abc... | EOA | N/A | Upgrade implementation | ❌ FAIL |

### 3.5.7 Aragon OTF Cross-Reference

After completing on-chain RPC verification, cross-reference findings with the [Aragon Ownership Token Framework](https://otf.aragon.org/) (OTF). OTF provides independent, evidence-backed governance analysis covering onchain control, value accrual, verifiability, and token distribution.

**How to use:**
1. Check if the protocol's governance token is covered at `https://otf.aragon.org/tokens/{symbol}`
2. Currently covers: AAVE, AERO, CRV, LDO, UNI, YB, SKY, ETHFI (and growing)
3. Compare OTF's governance classification against your RPC findings
4. OTF catches nuances that RPC calls miss:
   - **Offchain vs onchain voting** (e.g., EtherFi uses Snapshot + multisig, not binding onchain governance)
   - **L2 vs mainnet governance differences** (e.g., EtherFi L2 tokens upgradeable with no timelock)
   - **Token distribution concentration** (whale voting risk)
   - **IP and trademark control** (who actually owns the protocol brand)
   - **Value accrual mechanisms** (where fees flow)

**Red flags from OTF:**
- "Tokenholders do not have binding onchain control" — governance is advisory only, multisig can ignore votes
- "Upgradeable by multisigs with no timelock" on L2 deployments — even if mainnet is timelocked
- High insider allocation (>50% to team + investors) with unverified vesting
- Trademark/IP held by company not controlled by tokenholders

**Integration:** Use OTF as a SECOND OPINION after your RPC verification. If OTF and your on-chain findings disagree, investigate the discrepancy. OTF may know about governance structures (offchain voting, legal entities) that aren't visible on-chain.

### 3.5.8 Mint Rate Limit Verification

Even with proper multisig governance, check if privileged minting functions have rate limits configured. A protocol may have a mint rate limit function in the contract but configure it to allow infinite minting — rendering the safety mechanism useless.

**What to check:**
```bash
# Check if the token contract has rate limit functions
cast call <TOKEN_ADDRESS> "mintRateLimit()(uint256)" --rpc-url <RPC_URL>
cast call <TOKEN_ADDRESS> "mintCap()(uint256)" --rpc-url <RPC_URL>
cast call <TOKEN_ADDRESS> "maxMintAmount()(uint256)" --rpc-url <RPC_URL>

# Check if there's a per-block or per-period limit
cast call <TOKEN_ADDRESS> "mintPerBlock()(uint256)" --rpc-url <RPC_URL>
cast call <TOKEN_ADDRESS> "mintingLimit()(uint256)" --rpc-url <RPC_URL>

# For Aave-style protocols, check supply caps
cast call <POOL_CONFIGURATOR> "getReserveCaps(address)(uint256,uint256)" <ASSET> --rpc-url <RPC_URL>
```

**Red flags:**
- Rate limit function exists but is set to `type(uint256).max` or 0 (disabled)
- No rate limit function at all on mintable tokens
- Rate limit can be changed by same role that can mint (defeats the purpose)
- Rate limit set unreasonably high relative to TVL (e.g., rate limit > 10x current supply)

**Why this matters:**
If a privileged key is compromised (even from a multisig via social engineering or key extraction), a single transaction can mint unlimited tokens if there's no rate limit. A properly configured rate limit bounds the damage: even with a compromised key, the attacker can only mint X tokens per period. This buys time for detection and response.

The ideal setup: rate limit set to a reasonable multiple of expected daily minting, controlled by a DIFFERENT role than the minter, with a timelock on changes.

**Precedent:** ermin (Oct 2025) publicly called out a protocol whose mint rate limit function existed but was configured to allow infinite mint. The safety mechanism was there but turned off.

### 3.5.9 Historical Precedents (Why This Matters)
- **Resolv (March 2026):** $80M exploit via compromised EOA admin key that could infinite-mint their stablecoin
- **Gauntlet (March 2026):** Single EOA can unilaterally withdraw all funds from USD Alpha vault
- **Frax:** Timelock admin is a bare EOA — single key controls all governance
- **Fluid/Instadapp Arbitrum:** Proxy admin owner is EOA — can upgrade the entire Liquidity contract
- **EtherFi L2:** Mainnet has timelock, but L2 tokens upgradeable by multisig with NO timelock (caught by OTF cross-reference)
- **Camelot:** V4 factory owner is bare EOA, V3 is only 2/3 Safe — below minimum threshold
- **Infinite mint configs (Oct 2025):** Protocol had mint rate limit function but configured it to allow infinite mint — the safety mechanism existed but was disabled

---

## Phase 4: Risk Deep Dive

### 4.1 Systemic Risks
- What happens if the underlying protocol gets exploited?
- What is the maximum loss scenario for MYT depositors?
- Is there contagion risk? (does a failure here cascade to other strategies?)
- Smart contract risk score based on audit findings
- Oracle failure modes

### 4.2 Economic Risks
- Is the yield sustainable or subsidized? (incentive tokens that can dump)
- Token emission schedule for reward tokens
- What happens when incentives dry up?
- Impermanent loss risk (for LP strategies)
- Depeg risk for wrapped/synthetic assets

### 4.3 Operational Risks
- Can the protocol be paused? By whom?
- Governance attack vectors (low quorum, token concentration)
- Key management (multisig composition, threshold)
- Frontend/DNS risks
- Dependency on centralized services (RPCs, keepers, liquidators)

### 4.4 Regulatory Risks
- Is the protocol sanctioned or at risk of sanctions?
- Are any tokens involved securities/commodities concerns?
- KYC/AML compliance of the protocol
- Jurisdiction of the team (relevant for enforcement risk)

### 4.5 Inverse Finance Test
Apply the "Inverse test" — a protocol's incident RESPONSE pattern:
- How many incidents has this protocol had?
- Did they do a full architecture rewrite after incidents, or patch and continue?
- Is the current codebase a continuation of previously-exploited code?
- If >2 incidents: FLAG AS STRUCTURAL RISK regardless of other factors

---

## Phase 5: Kill Switch & Emergency Assessment

- How fast can the MYT killSwitch be activated?
- What is the multisig response time based on historical signing patterns?
- Can funds be recovered after killSwitch activation?
- What is the maximum damage window between incident detection and fund recovery?
- Does Hypernative or similar monitoring cover this protocol?

---

## Phase 6: Final Report

### Report Structure

```
# MYT Strategy Risk Assessment: [Protocol] [Strategy Name]
Date: [date]
Assessor: Jean (AI) — for internal risk team review

## Executive Summary
[2-3 sentences. What is it, what's the verdict, what's the biggest concern.]

## Strategy Overview
- Protocol: [name]
- Strategy: [type]  
- Chain: [chain]
- Tokens: [list]
- Expected yield: [X%]
- Yield source: [description]

## Compatibility Verdict
[PASS/FAIL on each Phase 2 check. Any FAIL = automatic REJECT.]

## Risk Classification Recommendation
[LOW / MEDIUM / HIGH / REJECT]

Rationale: [detailed reasoning with specific evidence]

## Key Findings

### Critical (must address before integration)
[numbered list]

### High (significant concern)
[numbered list]

### Medium (notable but manageable)
[numbered list]

### Low (minor concerns)
[numbered list]

## Protocol Scorecard

| Category | Score (1-10) | Notes |
|----------|-------------|-------|
| Team credibility | X | [brief] |
| Audit quality | X | [brief] |
| Time in production | X | [brief] |
| Incident history | X | [brief] |
| Smart contract quality | X | [brief] |
| Withdrawal reliability | X | [brief] |
| Yield sustainability | X | [brief] |
| Governance safety | X | [brief] |
| Oracle reliability | X | [brief] |
| Overall confidence | X | [brief] |

## Comparable Strategies
[How does this compare to strategies already in MYT?]

## Maximum Loss Scenario
[Worst case. How much could MYT depositors lose and under what circumstances.]

## Sources
[EVERY source must be a verified, clickable link. No dead links allowed.]
1. [Audit report — link]
2. [GitHub repo — link]
3. [Documentation — link]
4. [Incident reports — links]
5. [TVL data — link]
6. [Team info — link]
7. [Bug bounty — link]

## Recommendation for Risk Team
[Final recommendation. Remember: the default position is REJECT. 
Only recommend inclusion if the evidence overwhelmingly supports it.
Include specific conditions or monitoring requirements if recommending inclusion.]
```

### Risk Classification Criteria

**LOW risk (Tier 1):**
- 2+ years incident-free production operation
- TVL >$500M sustained
- Multiple high-quality audits (Trail of Bits, OpenZeppelin, Spearbit, etc.)
- Immutable or heavily timelocked contracts
- Active bug bounty >$500k
- Instant withdrawals with deep liquidity
- Score 8+ on all scorecard categories

**MEDIUM risk (Tier 2):**
- 6+ months production with no critical incidents
- TVL >$50M
- At least one quality audit
- Timelock >24h on admin operations
- Active bug bounty
- Reliable withdrawal history
- Score 6+ average, no category below 4

**HIGH risk (Tier 3):**
- Everything else that passes Phase 2 compatibility
- These strategies get lower allocation caps
- Require more frequent monitoring
- Higher killSwitch readiness

**REJECT:**
- Fails any Phase 2 compatibility check
- Fails any Phase 3.5 OpSec auto-fail condition (EOA/MPC controlling funds, upgrades, or minting)
- Any protocol with 3+ security incidents
- No audit at all
- EOA admin with no timelock
- Multisig threshold below 3/5 for fund-controlling roles
- Team fully anonymous with no track record
- Yield source is unsustainable incentive farming only
- Any evidence of prior rug pull or malicious behavior
- Proxy admin controlled by EOA (can upgrade implementation = can drain funds)
- Different governance quality across chains — REJECT the weaker chain deployment even if mainnet passes

---

## Output Requirements

1. **Save the full report** to `~/clawd/myt-assessments/[protocol]-[strategy]-[date].md`
2. **All source links must be verified** — click each one and confirm it loads
3. **Flag uncertainties explicitly** — if you can't verify something, say so
4. **The report is a STARTING POINT** — the human risk team makes the final call
5. **Never recommend a strategy you wouldn't put your own money in**
