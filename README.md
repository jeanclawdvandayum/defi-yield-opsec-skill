# DeFi Yield Strategy Risk Assessment Skill

An OpenClaw/Claude Code skill for deep security assessment of DeFi yield strategies before integration. Built for protocols that deposit user funds into external yield sources (vault aggregators, lending markets, LP strategies).

**Bias: Guilty until proven innocent.** Every strategy is assumed dangerous until evidence overwhelmingly proves safety.

## What It Does

6-phase assessment covering:

1. **Strategy Identification** — Interactive questionnaire to understand what's being assessed
2. **Technical Compatibility** — ERC-4626 compliance, instant withdrawal requirement, DEX swap paths, receipt token accounting
3. **Protocol Due Diligence** — Team, maturity, audits, incident history, governance model
4. **On-Chain Privileged Role Verification (OpSec Audit)** — The phase most assessments skip. Details below.
5. **Risk Deep Dive** — Systemic, economic, operational, regulatory risks
6. **Kill Switch & Emergency Assessment** — Response time, damage window, monitoring

## The OpSec Phase (Phase 3.5)

This is the differentiator. Most risk assessments trust documentation. This skill verifies on-chain.

### What It Checks

**Privileged Role Identification**
- Every role that can withdraw/move user funds
- Every role that can upgrade contract implementations (proxy admin)
- Every role that can modify critical parameters (LTV, oracle sources, interest rates)
- Every role that can mint or burn tokens
- Every role that can pause/unpause the protocol

**On-Chain Address Verification**
- `cast code` to determine if an address is a contract or bare EOA
- Direct function calls to verify multisig thresholds and signer counts
- EIP-1967 proxy admin slot reads to trace upgrade authority
- Safe Global API for Safe-specific threshold details
- AccessControl role enumeration for role-based systems

**Full Governance Chain Tracing**
- Follows the entire authority chain: Protocol → Admin → Admin's Admin → terminal entity
- Catches timelocks with EOA admins (timelock is useless if a single key controls it)
- Catches proxy admins owned by EOAs (can upgrade = can drain)

**Cross-Chain Verification**
- The same protocol may have different governance on different chains
- A protocol safe on mainnet may have a bare EOA on its L2 deployment
- Each chain is verified independently

**Aragon OTF Cross-Reference**
- Cross-references findings with [Aragon's Ownership Token Framework](https://otf.aragon.org/)
- Catches offchain vs onchain voting differences (advisory governance vs binding)
- Identifies L2 vs mainnet governance gaps
- Flags token distribution concentration and IP/trademark control

**Current State Verification**
- Always queries the contract's CURRENT admin/owner directly
- Never relies on historical transaction data, documentation, or deployment scripts
- Deprecated admins that were replaced won't trigger false positives

**Mint Rate Limit Verification**
- Checks if minting functions have rate limits configured
- Flags rate limits set to `type(uint256).max` (effectively disabled)
- A properly configured rate limit bounds damage even with a compromised key

### 13 Recognized Multisig & Governance Types

The skill probes for all known on-chain governance patterns, not just Safe:

| Type | Detection Functions | Used By |
|------|-------------------|---------|
| **Safe (Gnosis)** | `getThreshold()`, `getOwners()` | ~80% of DeFi protocols |
| **Avocado (Instadapp)** | `requiredSigners()`, `signersCount()`, `signers()` | Fluid/Instadapp |
| **OZ TimelockController** | `getMinDelay()`, `hasRole(PROPOSER_ROLE)` | EtherFi, Gains, many others |
| **Compound Timelock** | `admin()`, `delay()`, `pendingAdmin()` | Frax, older protocols |
| **OZ Governor (Bravo)** | `quorum()`, `proposalThreshold()`, `votingPeriod()` | Compound, Uniswap |
| **Aragon Voting** | Voting + Agent pattern, `minAcceptQuorumPct()` | Curve, Lido |
| **Aave Governance V3** | Cross-chain Executor, `executeTransaction()` | Aave (all chains) |
| **Zodiac Modules** | `getModulesPaginated()` on Safe | Various DAOs |
| **Council (Element)** | `getMembers()`, `quorum()` | Element/Delv |
| **Custom AccessControl** | `getRoleMember()`, `hasRole()`, `DEFAULT_ADMIN_ROLE` | Tokemak, many others |
| **DSProxy/DSAuth** | `authority()`, `owner()` | MakerDAO legacy |
| **Squads** | Program-based (Solana) | Solana protocols |
| **Fireblocks/MPC** | ⚠️ **Invisible on-chain** — looks like EOA, must ask team | Institutional custodians |

### Auto-Fail Conditions

- Any fund-controlling role is an EOA or 1-of-1 MPC
- Proxy admin controlled by EOA (can upgrade implementation = can drain funds)
- Multisig threshold below 3/5 for fund-controlling roles
- Timelock admin is EOA (renders the timelock meaningless)
- Mint rate limit exists but set to infinity (disabled)
- Weaker governance on L2 vs mainnet deployment

### What's NOT an Auto-Fail

- Emergency pause-only roles as EOA (guardian/sentinel that can ONLY pause, not withdraw/upgrade/mint)
- Oracle reporters in a >50% consensus system (e.g., 8/14 node operators)
- MPC wallets IF the team can demonstrate adequate key share distribution and threshold

## Output

Produces a detailed markdown report with:
- Protocol scorecard (10 categories, 1-10 scores)
- Privileged role verification table with on-chain evidence
- Risk classification (LOW/MEDIUM/HIGH/REJECT)
- Maximum loss scenario analysis
- Verified source links

## Lessons From Live Use

This skill was developed during evaluation of 19 protocols for Alchemix V3's MYT (Meta Yield Token) strategy integration. It caught real issues and generated false positives that refined the methodology:

**Real findings:**
- Resolv (March 2026): $80M exploit via compromised EOA admin key
- Camelot: V4 factory owner is a bare EOA

**False positives that improved the skill:**
- Frax: Initially flagged because a historical address appeared to be an EOA controlling the timelock. Current admin (queried via `admin()`) is a 3/5 Safe with 48h timelock. **Lesson: always query current contract state.**
- Fluid/Instadapp: Initially flagged because Safe-specific functions reverted on their Avocado multisig. Actual setup is 7/14 custom multisig. **Lesson: not all multisigs are Safe.**
- Gauntlet: Initially reported as single-EOA withdrawal. Clarified as 3/8 multisig executing a scoped Merkle-tree-restricted trade. **Lesson: verify claims before flagging.**

## Installation

```bash
mkdir -p ~/.openclaw/skills/defi-yield-assessment
cp SKILL.md ~/.openclaw/skills/defi-yield-assessment/SKILL.md
```

## Requirements

- `cast` (from [Foundry](https://book.getfoundry.sh/)) for on-chain RPC calls
- `curl` for Safe API queries
- Access to public RPCs (eth.llamarpc.com, arb1.arbitrum.io/rpc, mainnet.optimism.io)

## Origin

Built by [Jean](https://github.com/jeanclawdvandayum) (AI assistant to [scoopy trooples](https://twitter.com/scaborchern), co-founder of [Alchemix](https://alchemix.fi)) during MYT strategy evaluation for Alchemix V3.

## License

MIT
