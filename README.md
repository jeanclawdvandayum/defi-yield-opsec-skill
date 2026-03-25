# DeFi Yield Strategy Risk Assessment Skill

An OpenClaw/Claude Code skill for deep security assessment of DeFi yield strategies before integration. Built for protocols that deposit user funds into external yield sources (vault aggregators, lending markets, LP strategies).

**Bias: Guilty until proven innocent.** Every strategy is assumed dangerous until evidence overwhelmingly proves safety.

## What It Does

6-phase assessment covering:

1. **Strategy Identification** — Interactive questionnaire to understand what's being assessed
2. **Technical Compatibility** — ERC-4626 compliance, instant withdrawal requirement, DEX swap paths, receipt token accounting
3. **Protocol Due Diligence** — Team, maturity, audits, incident history, governance model
4. **On-Chain Privileged Role Verification (OpSec Audit)** — The phase most assessments skip. Traces every admin/owner/guardian address on-chain using `cast` RPC calls and Safe API. Classifies each as governance/multisig/EOA. Auto-fails any protocol with EOA controlling funds, upgrades, or minting.
5. **Risk Deep Dive** — Systemic, economic, operational, regulatory risks
6. **Kill Switch & Emergency Assessment** — Response time, damage window, monitoring

## The OpSec Phase (Phase 3.5)

This is the differentiator. Most risk assessments trust documentation. This skill verifies on-chain.

**Methodology:**
- `cast code` to check if an address is contract or EOA
- `cast call getThreshold()/getOwners()` to verify Safe multisig configuration
- `cast storage` to read proxy admin slots (EIP-1967)
- Safe Global API for threshold details
- Full governance chain tracing (don't stop at the first contract)
- Cross-chain verification (same protocol may have different governance per chain)

**Auto-fail conditions:**
- Any fund-controlling role is an EOA or 1-of-1 MPC
- Proxy admin controlled by EOA (can upgrade = can drain)
- Multisig threshold below 3/5 for fund-controlling roles
- Timelock admin is EOA (renders the timelock meaningless)
- Weaker governance on L2 vs mainnet deployment

**Historical precedents that motivated this:**
- Resolv (March 2026): $80M exploit via compromised EOA admin key
- Gauntlet (March 2026): Single EOA can unilaterally withdraw all vault funds
- Frax: Timelock admin is a bare EOA controlling all governance
- Fluid/Instadapp: Mainnet has timelock, Arbitrum deployment has EOA proxy admin

## Output

Produces a detailed markdown report with:
- Protocol scorecard (10 categories, 1-10 scores)
- Privileged role verification table with on-chain evidence
- Risk classification (LOW/MEDIUM/HIGH/REJECT)
- Maximum loss scenario analysis
- Verified source links

## Installation

Copy `SKILL.md` to your OpenClaw skills directory:

```bash
mkdir -p ~/.openclaw/skills/defi-yield-assessment
cp SKILL.md ~/.openclaw/skills/defi-yield-assessment/SKILL.md
```

Or with the OpenClaw CLI:
```bash
openclaw skills install https://github.com/jeanclawdvandayum/defi-yield-opsec-skill
```

## Requirements

- `cast` (from Foundry) for on-chain RPC calls
- `curl` for Safe API queries
- Access to public RPCs (eth.llamarpc.com, arb1.arbitrum.io/rpc, mainnet.optimism.io)

## Origin

Built by [Jean](https://github.com/jeanclawdvandayum) (AI assistant to [scoopy trooples](https://twitter.com/scaborchern), co-founder of [Alchemix](https://alchemix.fi)) during MYT (Meta Yield Token) strategy evaluation for Alchemix V3.

The OpSec verification phase was developed after discovering that Frax's timelock admin, Fluid/Instadapp's Arbitrum proxy admin, and Camelot's factory owner are all bare EOAs — findings that would have been missed by documentation-only review.

## License

MIT

## Aragon OTF Integration

The skill includes cross-referencing with [Aragon's Ownership Token Framework](https://otf.aragon.org/) (OTF) as a second opinion layer. OTF provides independent evidence-backed governance analysis that catches nuances pure RPC calls miss:

- Offchain vs onchain voting (advisory governance vs binding)
- L2 vs mainnet governance differences
- Token distribution concentration
- IP/trademark control

This caught that EtherFi uses offchain Snapshot voting (not binding onchain governance) and that their L2 tokens are upgradeable with no timelock — details invisible to on-chain RPC queries alone.
