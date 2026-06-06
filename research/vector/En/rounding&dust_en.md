# In-Depth Research Report on Real-World DeFi/NFTFi Rounding & Dust Incidents as of 2026-03-16

## Scope of Research and Root-Cause Family Definition

This report focuses on a vulnerability family that is often underestimated on-chain, but becomes systematically amplified in environments defined by composability and atomic multi-step execution: **integer division rounding down, precision truncation, incorrect rounding direction, and improper remainder ownership / dust cleanup policy**. These issues then create stable arbitrage opportunities or asset leakage across the pipeline of shares, accounting, exchange rates, and fees. Typical manifestations include:

When a protocol handles rounding in a way that is inconsistent with the safety posture that should favor the protocol, such as rounding inputs up, outputs down, and assigning remainders to the protocol, across paths including **scaling, price / exchange-rate conversion, share minting and burning, debt-share accounting, invariant solving, and batch settlement**, an attacker can use **repeated small operations, order splitting, or looping mint-burn-redeem / buy-sell / borrow-repay flows** to convert what should have been discarded error or protocol-absorbed dust into recurring profit.

This report follows your distinction requirements strictly:
- **Strict dust / rounding / truncation exploits**: incidents where rounding, truncation, or remainder handling directly creates systematic profit, usually together with accumulated micro-ops, extremely small `totalSupply` / `totalShares`, or batch amplification.
- **Different labels, same root-cause family**: incidents such as tick-calculation or double-counted liquidity bugs, solver instability that skews truncation, or arbitrage / asset loss caused by the interaction between accounting variables, branch conditions, and rounding direction.

## Methodology and Screening Criteria

- Only **publicly disclosed, real-world incidents** are included, meaning there is on-chain transaction / address evidence or authoritative post-incident confirmation. CTFs, demos, purely theoretical PoCs, and "bug bounty only, not exploited" cases are excluded.
- Priority is given to the high-quality sources you specified: **official postmortems / incident disclosures**, plus technical analysis from security teams such as **OpenZeppelin, Trail of Bits, BlockSec, SlowMist, and CertiK**. Wherever possible, this report also provides **transaction hashes, attacker contracts / addresses, and core contract addresses** as verifiable on-chain evidence.

## Highly Similar Case Set (Strict Dust / Rounding / Truncation Exploits)

The cases below are ordered by **similarity to the dust / rounding / accounting-mismatch root cause**, from highest to lowest. The earlier a case appears, the closer it is to the core pattern of repeated micro-operations accumulating dust / rounding error together with inconsistency across shares, `totalSupply`, or internal accounting.

### Balancer V2 Composable Stable Pools: `batchSwap` amplified rounding bias and led to losses above $120 million

**Classification**: strict rounding / precision loss / dust amplification exploit involving micro-swaps, batching, and internal-accounting amplification.

**Incident date**: 2025-11-03, with multiple analyses placing the start at around 07:40 UTC.

**Loss amount**: estimates range from **$120M+ to $128.64M**, depending on scope such as cross-chain exposure, forks, and later recovery / protection counts.

**Exploited function / module (critical path)**:
- `Vault.batchSwap` for atomic multi-step swaps, with internal accounting first and net settlement later
- `ComposableStablePool.onSwap` -> `_swapGivenOut` for the EXACT_OUT path -> `_upscale` / `_upscaleArray`, which scale balances by the `scalingFactor`
- `_scalingFactors`, where introducing exchange rates creates **non-unitary** scaling factors and makes truncation more exploitable
- Funds first accumulated as `InternalBalance` inside the Vault, then withdrawn via paths such as `manageUserBalance`, making this exploitable at the internal-accounting layer

**Root cause (mapped into the dust / rounding family)**:
- The scaling function `_upscale` or `_upscaleArray` uses **one-sided round-down (`mulDown`)** on a critical path, instead of distinguishing by swap direction between the safe rules of rounding input amounts up and output amounts down. When the amount is very small and the scaling factor is non-unitary, **precision truncation materially understates the amount owed and distorts the invariant and BPT price**, allowing a tiny bias to be amplified systematically across repeated iterations inside `batchSwap`.

**Attack path (high-level reconstruction)**:
The attacker used the "multi-step swap first, net settlement later" nature of `batchSwap` to turn a single swap's few-wei truncation bias into a reusable arbitrage engine:
1. First push the target pool into a **low-liquidity state with balances close to rounding boundaries**, making rounding error a larger relative factor.
2. Within one `batchSwap`, arrange dozens of micro-operations, with sources reporting 65+ steps, so that `_upscale`'s round-down bias accumulates inside invariant and price calculations.
3. Use the depressed implied BPT price to **acquire or redeem higher-value assets at a lower accounting cost**. Profit first appears in internal Vault accounting events such as `InternalBalanceChanged`, then gets withdrawn in later transactions.

**Attack transaction or attacker address (on-chain evidence)**:
The Check Point postmortem provides a relatively complete three-party structure of deployer / exploit contract / recipient, together with the two key transactions, one for deployment and theft and one for withdrawal.

**Main reference sources**:
- Check Point Research postmortem, including transactions, addresses, code path, and internal-balance perspective
- OpenZeppelin technical analysis, explaining `_upscale`, `_scalingFactors`, the `batchSwap` call chain, and why "very small amount + non-unitary scaling factor" amplifies truncation
- Trail of Bits summary and audit perspective, emphasizing the long-term nature of the rounding-direction issue and the need for testing and invariant strategies
- BlockSec deep dive, emphasizing rounding inconsistency and the core attack flow

**Would the exploit still work if rounding direction / remainder ownership / dust policy were designed correctly?**
Most likely **no, or at least not at scale**. If `_upscale` rounded up or down depending on swap direction, and the protocol enforced constraints for small-amount / low-liquidity states such as minimum swap sizes, upper bounds on rounding bias, invariant-delta validation, and prevention of cumulative bias within one batch, this kind of stable micro-error arbitrage would be cut off directly.

**Original links (for verification)**:
```text
Check Point postmortem:
https://research.checkpoint.com/2025/how-an-attacker-drained-128m-from-balancer-through-rounding-error-exploitation/

Key transactions (listed in the article):
TX1 (deploy+theft): 0x6ed07db1a9fe5c0794d44cd36081d6a6df103fab868cdd75d581e3bd23bc9742
TX2 (withdraw):     0xd155207261712c35fa3d472ed1e51bfcd816e616dd4f517fa5959836f5b48569

Relevant attacker addresses (listed in the article):
Deployer:         0x506D1f9EFe24f0d47853aDca907EB8d89AE03207
Exploit Contract: 0x54B53503c0e2173Df29f8da735fBd45Ee8aBa30d
Recipient:        0xAa760D53541d8390074c61DEFeaba314675b8e3f

OpenZeppelin technical analysis:
https://www.openzeppelin.com/news/understanding-the-balancer-v2-exploit

Trail of Bits analysis and guidance:
https://blog.trailofbits.com/2025/11/07/balancer-hack-analysis-and-guidance-for-the-defi-ecosystem/
```

### Yearn yETH: composite infinite mint via solver truncation to zero, dust-triggered initialization branch, and underflow

**Classification**: strict truncation / rounding + dust + accounting-variable inconsistency, as a multi-stage composite exploit.

**Incident date**: 2025-11-30, at block 23,914,086.

**Loss amount**: the official disclosure states that multiple LST assets in the pool were drained and notes that 857.49 pxETH was later recovered with assistance; outside reporting often summarizes the loss at roughly the $9M level, depending on counting methodology.

**Exploited function / module (critical path)**:
- `add_liquidity` / `remove_liquidity`, for LP entry / exit and supply / invariant updates
- `_calc_supply`, an iterative fixed-point solver using Newton-style approximation
- `update_rates` / `_update_supply`, which reconcile internal `D` with ERC-20 `totalSupply`, and also involve the `st-yETH` POL mechanism that absorbs losses / gains

**Root cause (mapped into the dust / rounding family)**:
The official disclosure breaks the exploit into three phases. The parts most relevant to this report are:
- In Phase 1, solver iteration became numerically unstable under extremely imbalanced input, and **`sp / s` rounded down to 0 under integer arithmetic**, collapsing the product accumulator `r` to 0. That in turn caused `Π = 0`, degraded the invariant, and enabled excess minting.
- In Phase 3, after the pool had been emptied and `prev_supply == 0`, the attacker re-entered the initialization branch using a dust configuration and then triggered an arithmetic underflow similar to `unsafe_sub(A*Σ, D*Π)`, resulting in a massive LP mint. This is a combined boundary-branch + dust + arithmetic-safety issue.

**Attack path (summary of the official three-phase narrative)**:
1. **Phase 1**: construct an extremely imbalanced `add_liquidity` sequence that drives the solver outside its intended domain. During iteration, integer truncation makes a key ratio round down to 0, `r -> 0`, then `Π -> 0`, and because the protocol lacks adequate domain validation, it accepts an incorrect `D` and over-mints LP.
2. **Phase 2**: call paths such as `remove_liquidity(0)` to "repair" `Π` without correcting the inflated `D`; then use permissionless `update_rates` to trigger `_update_supply`, treating the discrepancy as system slashing and burning `st-yETH` held as POL to reconcile supply, thereby **shifting the cost of over-minting onto POL rather than the attacker**. Repeating this loop allows the attacker to extract LSTs.
3. **Phase 3**: after draining the pool, use dust to re-enter the initialization branch and trigger underflow, minting roughly `2.3544 x 10^56` LP and then draining the downstream yETH/ETH pool as well.

**Attack transaction or attacker address (on-chain evidence)**:
The official disclosure provides the exploit transaction and contract references, including a BlockSec Explorer link.

**Main reference sources**:
Yearn security disclosure, which is official and includes phase breakdown, contract / transaction references, and timeline.

**Would the exploit still work if rounding direction / remainder ownership / dust policy were designed correctly?**
Most likely **no**. If the protocol enforced strong on-chain validation of the solver domain and convergence conditions, avoided allowing key ratios to collapse to zero under integer truncation, and imposed strict constraints on state and parameters when entering initialization branches such as `prev_supply == 0`, while also using safe arithmetic to block underflow, the attack chain would be cut off in Phase 1 or Phase 3.

**Original links (for verification)**:
```text
Yearn official incident disclosure (with contract and transaction references):
https://github.com/yearn/yearn-security/blob/master/disclosures/2025-12-01.md

yETH Weighted Stableswap Pool contract (referenced in the disclosure):
https://etherscan.io/address/0xCcd04073f4BdC4510927ea9Ba350875C3c65BF81

Exploit transaction (BlockSec Explorer link referenced in the disclosure):
https://app.blocksec.com/explorer/tx/eth/0x53fe7ef190c34d810c50fb66f0fc65a1ceedc10309cf4b4013d64042a0331156
```

### Hundred Finance: draining a market with "2 wei totalSupply + donation-manipulated exchangeRate + truncate round-down"

**Classification**: strict dust exploit combining extremely small `totalSupply`, rounding down, and exchange-rate manipulation via an empty-market donation.

**Incident date**: 2023-04-16.

**Loss amount**: about **$7.4M**.

**Exploited function / module (critical path)**:
- `mint()` / `redeem()` in Compound v2 fork-style hToken / soToken logic
- `exchangeRate` calculation, involving variables such as `getCash()` and `totalSupply()`
- `truncate()` or floor-style integer conversion causing precision loss, essentially the pattern where "the shares that should be burned are just below an integer threshold, so after floor conversion 1 unit less gets burned"

**Root cause (mapped into the dust / rounding family)**:
BlockSec explains that the core issue is in `redeem()`: the theoretical hToken amount that should have been deducted was close to 2, for example 1.99999992, but because `truncate()` or integer conversion uses **rounding down**, only 1 was actually deducted. This created a systematically exploitable "under-burn" bias. When the market is in an **empty or low-liquidity state, especially if `totalSupply` is deliberately pushed close to 0**, the attacker can further manipulate `getCash()` by donating funds directly to the market contract, blowing up the exchange rate so that this single unit of under-burn corresponds to a large amount of asset value.

**Attack path (summarized from the BlockSec postmortem)**:
After preparing funds through a flash loan, the attacker intentionally reduced the hWBTC market's `totalSupply` to **2 wei** by minting some tokens and then redeeming until only a tiny remainder was left. The attacker then **transferred a large amount of WBTC directly into the market contract** without going through mint, so hWBTC supply did not increase while `getCash` became huge and `totalSupply` stayed tiny, pushing the exchange rate up by hundreds of times. Finally, the attacker exploited truncation in share deduction during redemption, turning the "should burn 2 but only burns 1" discrepancy into realizable profit and then extended the attack to drain other markets.

**Attack transaction or attacker address (on-chain evidence)**:
BlockSec provides the attack transaction hash on Optimism.

**Main reference sources**:
The BlockSec incident analysis, which includes root-cause explanation and a step-by-step flow.

**Would the exploit still work if rounding direction / remainder ownership / dust policy were designed correctly?**
Most likely **no**. If redemption were calculated on the protocol-safe side, for example by rounding the required burned shares up, or by constraining extreme exchange-rate conditions, and if empty-market states were eliminated through seeded liquidity, a minimum nonzero `totalSupply`, or correct handling / prohibition of donation effects, the attack surface would disappear.

**Original links (for verification)**:
```text
BlockSec postmortem:
https://blocksec.com/blog/6-hundred-finance-incident-catalyzing-the-wave-of-precision-related-exploits-in-vulnerable-forked-protocols

Attack transaction (hash given by BlockSec, on Optimism):
0x6e9ebcdebbabda04fa9f2e3bc21ea8b2e4fb4bf4f4670cb8483e2f0b2604f451

Reference link:
https://optimistic.etherscan.io/tx/0x6e9ebcdebbabda04fa9f2e3bc21ea8b2e4fb4bf4f4670cb8483e2f0b2604f451
```

### Sonne Finance: a Compound fork replay of "donation + tiny supply + truncation" causing about $20M in losses

**Classification**: strict precision loss / truncation plus empty-market donation manipulation, from the same exploit lineage as Hundred Finance.

**Incident date**: 2024-05-14. Multiple sources use this date for the main exploit, and some continued tracing the fund flows on 2024-05-15.

**Loss amount**: about **$20M**.

**Exploited function / module (critical path)**:
- Empty-market and tiny-`totalSupply` `exchangeRate` logic during market creation / initialization in a Compound v2 fork `soToken` market
- `borrow()`, `getAccountSnapshot()`, and truncation bias in collateral calculations
- Direct donation, meaning assets are transferred into the `soToken` contract, increasing `totalCash` without minting additional `soToken`

**Root cause (mapped into the dust / rounding family)**:
CertiK explicitly classifies the Sonne incident as a reuse of a **known precision-loss bug, the same family as Hundred Finance, against an empty or newly created market**.
SharkTeam further explains the mechanism: the attacker used a flash loan to transfer a large amount of VELO **directly into the `soVELO` contract**. Because this bypassed minting, `soVELO` `totalSupply` remained unchanged while `totalCash` surged, pushing the exchange rate sharply upward. The attacker then exploited **truncation** in the borrow and redeem paths, causing collateral requirements to be understated and share-conversion calculations to become biased, allowing a tiny amount of `soVELO` to support large borrowing and redemption.

**Attack path (summary from the SharkTeam reconstruction)**:
1. Use a flash loan to obtain about 35,569,150 VELO and transfer it directly into `soVELO` as a donation, making `totalCash` increase while `totalSupply` stays unchanged.
2. Deploy or use an attacker contract, post a small amount of `soVELO` as collateral, enable `soWETH` / `soVELO` as collateral and borrowing markets, and borrow WETH.
3. Exploit truncation so that the collateral or share amount that should be required is understated, while also redeeming back the large amount of VELO previously donated; repeat to amplify profit.
4. Repay the flash loan and keep the remaining assets as profit.

**Attack transaction or attacker address (on-chain evidence)**:
SharkTeam provides the attack transaction hash.

**Main reference sources**:
- CertiK incident analysis, which classifies it inside the known precision-loss / Compound-fork family
- SharkTeam transaction-level postmortem, which lays out the donation -> borrow -> repeat path and provides the transaction hash
- Halborn's writeup, which emphasizes how a stepwise, permissionless execution path made this class of donation attack exploitable

**Would the exploit still work if rounding direction / remainder ownership / dust policy were designed correctly?**
Most likely **no**. If new markets were forced to launch with sufficient seeded supply to avoid dust-level `totalSupply`, if donations either minted corresponding shares or were disallowed entirely, and if collateral, conversion, and borrow paths rounded in a protocol-safe way, rounding collateral demand up and borrowable amounts down, the exploit chain would be broken.

**Original links (for verification)**:
```text
CertiK analysis:
https://www.certik.com/blog/sonne-finance-incident-analysis

SharkTeam analysis (including attack transaction hash):
https://medium.com/@sharkteam/sharkteam-analysis-of-sonne-finance-attack-incident-b57a4529c475
Attack transaction (appears in the article):
0x9312ae377d7ebdf3c7c3a86f80514878deb5df51aad38b6191d55db53e42b7f0

Halborn postmortem:
https://www.halborn.com/blog/post/explained-the-sonne-finance-hack-may-2024
```

### Raft: inflated index plus `divUp` rounding made "1 wei collateral mints 1 wei share" repeatable and sufficient to support large borrowing

**Classification**: strict precision / rounding issue in share minting, plus amplification through repeated tiny operations, with a business-logic flaw that inflated the index.

**Incident date**: 2023-11-10. The official postmortem gives 18:59:23 UTC.

**Loss amount**: the official description says roughly **$6.7M of unbacked R** was minted and sold, causing depegging. Some analyses instead describe the scale in terms of ETH obtained and note the later twist where the attacker accidentally burned ETH.

**Exploited function / module (critical path)**:
- `InterestRatePositionManager` (IRPM), explicitly identified in the official writeup as the exploited contract
- `divUp` round-up behavior in share minting, which let an extremely small input still mint 1 unit of shares
- The index (`index` / `storedIndex`) being abnormally amplified through donation and liquidation paths, increasing the value represented by a single share

**Root cause (mapped into the dust / rounding family)**:
The official postmortem describes the main issue as **a precision-calculation problem during share-token minting** that granted the attacker extra shares. Once the index had been inflated, the attacker could exchange a tiny amount of shares for a large amount of cbETH, and then use the inflated collateral power to borrow a large amount of R.

**Attack path (step-by-step from the official writeup)**:
1. Use a flash loan to obtain 6,000 cbETH.
2. Transfer a total of 6,001 cbETH into IRPM and trigger liquidation on a pre-created position.
3. Inflate the collateral index by thousands of times through donation.
4. Exploit `divUp`: use **1 wei of cbETH to mint 1 wei of shares**, repeating this 60 times so that dust-scale inputs accumulate into meaningful collateral representation.
5. Redeem a large amount of cbETH with a very small amount of `rcbETH-c`, then use that collateral position to borrow roughly 6.7 million R, and swap it for ETH and other assets across multiple pools. The postmortem also records the attacker's accidental burn of 1,570 ETH.

**Attack transaction or attacker address (on-chain evidence)**:
The official postmortem provides the exploit transaction and attacker address.

**Main reference sources**:
The official Raft postmortem, which includes transactions, addresses, steps, and root cause, plus the summary index by DN Institute.

**Would the exploit still work if rounding direction / remainder ownership / dust policy were designed correctly?**
Most likely **no**. If share minting handled tiny inputs using protocol-safe rounding, usually floor rounding, a minimum mint threshold, or a minimum collateral unit, and if the index could not be blown up to an economically unreasonable range through donation or liquidation paths, the repeated "1 wei mints shares" loop would not turn into redeemable value.

**Original links (for verification)**:
```text
Raft official postmortem (with exploit tx / attacker / contract):
https://paragraph.com/%40raft/raft-security-incident-post-mortem-analysis-and-recovery-plan

Exploit transaction (given in the official article):
https://etherscan.io/tx/0xfeedbf51b4e2338e38171f6e19501327294ab1907ab44cfd2d7e7336c975ace7

Attacker (given in the official article):
https://etherscan.io/address/0xc1f2b71a502b551a65eee9c96318afdd5fd439fa

Exploited contract (given in the official article):
https://etherscan.io/address/0x9ab6b21cdf116f611110b048987e58894786c244
```

### Abracadabra: rebase accounting diverged at the `elastic=0, base!=0` boundary, and looping borrow / repay underestimated debt through rounding bias

**Classification**: strict fee / debt-accounting rounding mismatch. It does not look like a classic dust exploit on the surface, but at root it is still an accounting-variable mismatch caused by rounding and boundary-state handling.

**Incident date**: 2024-01-30.

**Loss amount**: around **$6.4M-$6.5M**, with slight variation by media source and accounting method.

**Exploited function / module (critical path)**:
- `Cauldron v4`, which multiple reports identify as the affected family
- The rebase-library accounting logic that keeps `elastic` and `base` in sync; the bug appears when **`elastic == 0` but `base` is nonzero**, and the discrepancy is not handled correctly, leaving a rounding gap that can be exploited through repeated operations
- Reports mention the attacker repeatedly calling `userBorrowPart()` together with `repay()` loops, matching the repeated-small-operations pattern you wanted to emphasize

**Root cause (mapped into the dust / rounding family)**:
As cited by Blockworks, the intended synchronization between `elastic` and `base` failed under a boundary condition. By repeatedly exploiting the resulting rounding error, the attacker was able to make the system underestimate debt, creating structural profit where more was borrowed than the system later counted as owed.

**Attack path (stable portion that can be reconstructed from public information)**:
1. In a specific Cauldron v4 instance, trigger the rebase synchronization flaw through repeated borrow / repay loops.
2. Because `elastic` and `base` diverge under certain states, and the logic does not properly absorb or isolate remainders, the system gradually underestimates debt.
3. Extract net MIM out of the system, then swap it on-chain into ETH and other assets.

**Attack transaction or attacker address (on-chain evidence)**:
Blockworks points to a "suspect transaction" whose hash appears in the report, although the chain-level details still need to be checked directly in an explorer.

**Main reference sources**:
- Blockworks reporting, which explicitly mentions a rounding bug, the rebase library, the `elastic=0, base!=0` condition, and a traceable transaction
- The Block's report, which provides the roughly $6.4M loss figure
- Other aggregation reports citing CertiK's assessment of a rounding issue plus borrow / repay loop, used here to reinforce the attack-chain description

**Would the exploit still work if rounding direction / remainder ownership / dust policy were designed correctly?**
Most likely **no**. If the rebase and debt-share accounting strictly enforced invariants such as `base == 0` iff `elastic == 0`, or otherwise implemented an exact conversion rule, if remainders were systematically assigned to the protocol or accumulated in a dedicated remainder account, and if repeated borrow / repay activity were constrained by minimum units, frequency limits, or state-machine rules, the looping amplification would stop working.

**Original links (for verification)**:
```text
Blockworks report (including the "suspect transaction" reference):
https://blockworks.com/news/rounding-exploit-magic-internet-money

The Block report:
https://www.theblock.co/post/275072/abracadabra-finance-drained-of-estimated-6-4-million-in-apparent-security-attack
```

## Same Root-Cause Family, Different Labels

### KyberSwap Elastic: incorrect rounding direction shifted tick calculation and double-counted liquidity, enabling drain through a flash-loan path

**Classification**: same-family event driven by a rounding-direction flaw, although the attack surface looks more like a tick / liquidity-mechanism issue than a classic dust exploit. At root it is still **wrong rounding direction -> state-judgment drift -> double-counted accounting / liquidity**.

**Incident date**: 2023-11-23.

**Loss amount**:
- BlockSec summary figure: **over $48M**
- SlowMist / MistTrack figure: **over $54.7M** across chains
The gap mainly comes from different counting scopes, cross-chain splits, and later tracing methods.

**Exploited function / module (critical path)**:
- `computeSwapStep`, the core math step inside the swap while-loop
- `estimateIncrementalLiquidity` / `deltaL` calculation, where the comment semantics expect `deltaL` to round up so that `nextSqrtP` rounds down, but the implementation mistakenly uses `mulDivFloor`, causing **`deltaL` to round down** and `nextSqrtP` to round up incorrectly
- The incorrect `nextSqrtP` and tick-transition judgment then create a state where the tick is considered not crossed even though price has already moved out of range, resulting in **double-counted liquidity** and profitable reverse swaps

**Root cause (why it belongs to the same family)**:
The exploitability of this incident did not come from classic reentrancy or oracle manipulation. It came from **a mismatch between mathematical implementation and the rounding direction implied by the comments and intended logic**. The attacker constructed an extremely precise input where a one-unit rounding difference was enough to change the tick / liquidity update branch, ultimately turning that tiny discrepancy into double-counted liquidity and redeemable arbitrage.

**Attack path (BlockSec six-step summary)**:
The attacker used a flash loan to borrow WETH and push price into a region with no base liquidity but existing reinvest liquidity. They then added and removed liquidity to tune the range and total amount, and finally executed a swap that tricked the pool into believing the tick had not crossed even though `sqrtP` had already crossed the next boundary. After that, a reverse swap realized profit because liquidity had been double-counted. This is a textbook case where one rounding difference becomes economic gain.

**Attack transaction or attacker address (on-chain evidence)**:
- A HackMD summary provides a representative attack transaction, attacker address, attacker contract, and affected pool contract
- SlowMist provides a list of attacker addresses from a cross-chain tracing perspective

**Main reference sources**:
BlockSec's technical postmortem and SlowMist's fund-tracing analysis.

**Would the exploit still work if rounding direction / remainder ownership / dust policy were designed correctly?**
Most likely **no**. If `deltaL` were rounded according to the intended comment semantics, namely ceil, and if the "tick not crossed" branch strictly enforced that `nextSqrtP` cannot exceed the next tick's `sqrtP`, while also preventing any chance of double-counted liquidity during state updates, the key turning point of the exploit would disappear.

**Original links (for verification)**:
```text
BlockSec postmortem (summary version):
https://blocksec.com/blog/kyberswap-incident-masterful-exploitation-of-rounding-errors-with-exceedingly-subtle-calculations

SlowMist / MistTrack postmortem (including multiple attacker addresses):
https://slowmist.medium.com/a-deep-dive-into-the-kyberswap-hack-3e13f3305d3a

HackMD summary (including representative attack tx / attacker / pool contract):
https://hackmd.io/@nomorecaffeine/BkHxa9Ta1e
Attack transaction (appears in the article):
https://etherscan.io/tx/0x485e08dc2b6a4b3aeadcb89c3d18a37666dc7d9424961a2091d6b3696792f0f3
Attacker address (appears in the article):
0x50275E0B7261559cE1644014d4b78D4AA63BE836
```

## Scenario Match and Search Result

As of this public-source search cutoff on **2026-03-16**, I **did not find** any **fractional NFT market settlement / NFTFi shard redemption / order-matching partial-fill rounding** exploit that simultaneously meets all of your conditions:
- it must be a real incident,
- it must have strong postmortem coverage and on-chain evidence,
- and its core root cause must clearly fall under rounding / dust / improper remainder handling,

with evidence strength and detail completeness comparable to the DeFi cases above.

This does not mean NFTFi lacks this risk. A more accurate interpretation is:
- public high-loss NFTFi incidents are more often centered on approvals / signatures, collateral-bypass flaws, oracle or floor-price manipulation, liquidation logic, and reentrancy / callback issues;
- in NFTFi, the dust / rounding family more often appears as **small leakage or boundary-condition bugs**, and if the loss was not large, teams may have handled it through patches or reimbursement instead of a detailed public postmortem, leaving a weaker public evidence chain than in major DeFi AMM or lending incidents.

If you want to push the search surface closer to the fractional-NFT theme, the best adjustment would be to relax the condition from "must involve a large loss" to "must have caused an actual accounting error or excess claim, even if the amount was small", and expand the target set to include:
- consistency between `previewRedeem` and `redeem` in fractional NFT redemption contracts
- remainder allocation in auction or matching-engine partial fills
- royalty or fee-distribution remainder accumulation and withdrawability boundaries

That said, doing so would push against your current requirement boundary, because many such cases become "bug discovered and fixed" rather than "real exploited attack."

## Defensive Takeaways and Practical Triage Criteria Abstracted from Real Incidents

These incidents point to a common engineering reality: in DeFi and NFTFi, **rounding is not an implementation detail, but an economic security boundary**.

The following practical defenses map directly back to the root-cause patterns you listed, and every one of them is supported by at least one real incident above:

Rounding direction must be tied to asset flow, rather than treating "always round down" as safe. The Balancer case shows that when scaling factors incorporate exchange rates and become non-unitary, blanket round-down can systematically understate payment obligations on paths such as EXACT_OUT, and batch or micro-operations can then amplify the bias.

Protocols must explicitly define remainder ownership and dust cleanup policy, and ensure dust is zeroed out or converged away at state-transition boundaries. The Yearn yETH incident shows that when the system enters initialization or reset branches such as `prev_supply == 0`, allowing dust to reach that branch, or leaving underflow exposure there, can turn a boundary state into an infinite-mint entry point.

Reachable states where `totalSupply` or `totalShares` are extremely small or zero should be eliminated, or guarded by provable safety invariants. Compound-fork incidents like Hundred and Sonne repeatedly show that **empty market + donation + truncation** turns tiny supply into a lever for exchange-rate manipulation.

Repeated tiny operations need explicit economic boundaries: minimum mint / redeem unit, minimum swap input / output, maximum iteration count, and a cumulative remainder ledger. Both Raft and Abracadabra show that when one-wei-scale operations can change shares or debt shares, attackers can loop the resulting error into real assets.

Atomic multi-step entry points such as `batchSwap`, `cook`, or multi-action routers must be tested from a **composed attack** perspective. A single step being correct does not imply the composition is safe. Balancer and KyberSwap are both examples where each individual step looks like a tiny error, but the combined sequence reaches a new state space and becomes profitable.

Finally, a highly practical triage rule is this: if you can construct a path where the system ever reaches **"should deduct 2 units but only deducts 1"**, or **"should be zero but is not zero"**, or a **one-sided accumulated rounding bias**, and that bias can be amplified through looping or batching, then it almost certainly belongs to the family you described and should be prioritized as a systemically exploitable arbitrage risk.
