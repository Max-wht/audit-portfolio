# Sensitive On-Chain Data / Public Storage Disclosure: An In-Depth Security Research Report

## Executive Summary

"Sensitive On-Chain Data / Public Storage Disclosure" fundamentally refers to **writing information that should remain secret into a public blockchain in a recoverable form** such as storage, calldata, event logs, or historical transactions, or mistakenly assuming that Solidity `private`, unverified source code, or complicated storage layouts can provide confidentiality. In practice, attackers can reconstruct such secrets from public on-chain data and then use them to bypass access control, manipulate game mechanics, or directly steal assets. SWC classifies this misconception under **SWC-136 (Unencrypted Private Data On-Chain)**, emphasizing that even when source code is not public, attackers can still infer state values from on-chain transactions and storage behavior.

As of **March 16, 2026**, public evidence suggests that **strict, large-scale real-world exploits caused purely by putting a secret on-chain and having it read back are relatively uncommon**. Instead, the pattern appears more often in three forms:  
1) **classic teaching or CTF examples** such as reading a storage slot to recover a `password`; 2) **small games, gambling, or lottery contracts** that expose answers, moves, or winning factors in calldata or storage before settlement; and 3) **major boundary cases** that are usually classified under randomness, commit-reveal, or MEV, even though the deeper issue is still that public blockchain information was mistakenly treated as secret or safely reusable.  

The main reason strict large-loss cases are rare is straightforward: the industry already broadly understands that **public chains cannot hide state from the network**. Mature projects usually do not place critical keys, passwords, lottery answers, or other secrecy-dependent material in plaintext on-chain. Even when such mistakes exist, they are often caught during audit or testing before they develop into large on-chain losses.

## Search Strategy and Classification Criteria

This report uses **March 16, 2026 (Asia/Taipei)** as its cutoff date. Events occurring after that date are excluded from the main body. The research approach follows a multilingual, multi-source, cross-validated method covering both English and Chinese materials, and attempts to provide, wherever possible, a combination of **on-chain evidence (contract addresses and transactions)** and **interpretive analysis (postmortems and research articles)**.

**Reusable search strategy**  
- Terminology expansion: searches covered terms such as *secret, password, salt, private variable, read storage slot, eth_getStorageAt, calldata leak,* and *broken commit-reveal*, rather than relying on a single phrase like "private variable leak."  
- Bilingual parallel search: English sources focused on SWC, official docs, and professional security writeups; Chinese sources focused on security communities, audit checklists, Top 10 vulnerability summaries, and translated Solidity security references.  
- Evidence priority: primary sources first (project disclosures, block explorers, official documentation), then secondary analysis (security firms and researchers), and finally explicit inference based on public evidence when necessary.

**Classification rules for the three lists**  
- **Core included cases (A)**:  
  1) the key vulnerability mechanism is directly tied to sensitive information being placed on-chain and publicly readable;  
  2) the attacker actually recovered information that was meant to remain secret from public chain data such as storage, calldata, logs, or transaction history;  
  3) recovering that information was a necessary or core step in the exploit;  
  4) there is clear evidence of real exploitation and impact, such as loss of funds, broken access control, or a mechanism being genuinely compromised.  
- **Boundary cases (B)**: public on-chain information was used in the exploit, but the case is not a pure "secret disclosure" case. Typical examples include predictable randomness, flawed commit-reveal designs, mempool or calldata replay/front-running, or cases mixed with access control, signature, or business logic flaws. These should be discussed separately and clearly justified as boundary cases.  
- **Excluded cases (C)**: leaked private keys or seed phrases, backend breaches, frontend compromise, pure reentrancy, flash loan exploits, oracle manipulation, and unrelated front-running do not belong to the core sample.

## Vulnerability Definition and Boundaries

**Formal definition**  
This report defines "Sensitive On-Chain Data / Public Storage Disclosure" as follows:  
> A contract or application places material that should remain secret, such as passwords, answers, salts, seeds, signature material, or gatekeeping credentials, directly or reversibly into publicly readable on-chain surfaces including storage, calldata, events, and historical transactions, or incorrectly relies on `private`, unverified source code, structural complexity, or slot obscurity as a confidentiality mechanism, allowing an adversary to reconstruct the secret and profit from it.

SWC-136 explicitly states that a `private` variable does **not** mean "unreadable off-chain." Even if the contract source is not published, an attacker can still derive state values from the observable chain. As a result, unencrypted sensitive data should never be treated as safe merely because it resides in contract code or state.

**Boundaries relative to neighboring vulnerability classes**  
- **Versus predictable randomness (SWC-120)**: randomness bugs often treat on-chain visible values such as `blockhash` or `timestamp` as entropy. The core issue there is that the entropy source is predictable or manipulable, not necessarily that a secret value was stored and later read out. Such cases usually belong in boundary category B, even though they share the same underlying misconception: treating public chain data as if it were hidden or safe from adversarial use.  
- **Versus broken commit-reveal**: commit-reveal is supposed to defer disclosure by separating commitment from revelation. If implemented poorly, for example with low-entropy secrets, unbound reveals, or reveal transactions that can be copied from the mempool, the exploit may operate at the mempool or historical calldata layer. These are boundary or mixed-root-cause cases rather than the purest form of on-chain secret disclosure.  
- **Versus frontend or backend leaks**: if the secret comes from GitHub, a database, frontend code, social engineering, or operational compromise, then the root cause is not public chain data disclosure and the case should be excluded.  
- **Versus ordinary mempool front-running**: if a transaction is front-run merely because it is visible in the mempool, such as in common AMM sandwich attacks, that is not this class of issue. But if what gets copied is a reveal preimage or a reusable credential, then it becomes a boundary case.  
- **Versus plain access control bugs**: if the exploit succeeds solely because of missing `onlyOwner` checks or absent authorization validation, it does not belong here. It only falls into this topic if the key material used to bypass the control became available because the chain exposed it publicly.

## EVM and Solidity Technical Background

**Why storage is inherently public on-chain**  
Contract state is stored in blockchain storage, and every node must be able to read those values in order to execute transactions and reach consensus. Therefore, **there is no native mechanism to hide storage from the network on a public chain**. If something must remain secret, it has to stay off-chain, or at minimum be encrypted on-chain with decryption keys kept away from publicly readable chain surfaces. Solidity's security guidance explicitly warns that private information and randomness cannot rely on on-chain state confidentiality.

**`private` is not encryption, only visibility**  
In Solidity, `private` and `internal` only restrict direct access at the language level for other contracts. They do not prevent off-chain observers from reading storage. SWC-136 treats the mistaken belief that `private` means secret as a textbook risk.

**How slots, packing, mappings, dynamic arrays, and structs are read**  
- **Slot order and packing**: Solidity maps state variables to consecutive storage slots in declaration order, starting from slot 0, and may pack smaller types into the same slot.  
- **Mapping slot calculation**: the location of a mapping entry is computed from the key and the base slot using a `keccak256`-based layout. Nested mappings and structs require additional offsets. Solidity documentation provides the exact formulas and examples.  
- **Dynamic arrays**: the slot itself stores the length, while the data area begins at `keccak256(slot)`. Elements are then laid out from that base according to element width.

**How attackers actually obtain this data**  
- Through standard JSON-RPC: `eth_getStorageAt` can read a specific storage slot for a target contract at a given block height.  
- Through infrastructure providers: many node and API providers document the same call for auditing and contract analysis use cases.  
- Through block explorers: explorers expose input data, event logs, and interaction history, which can help infer where a supposed secret was written during deployment or function execution.

## Case Lists and Detailed Analysis

The report first presents a **structured included/boundary/excluded list**, then a consolidated case table, and finally deeper discussion of representative incidents. Because strict A-class large-loss cases are rare in public evidence, the table explicitly labels teaching examples and explains why boundary cases are not treated as pure secret-disclosure cases.

### Case Table

> Notes:  
> - "Core on-chain sensitive data case" is marked "Yes" only when the case meets the A-class definition.  
> - "Primary evidence sources" ideally include at least two sources, commonly a block explorer or on-chain artifact plus a postmortem or research article.  
> - "Confidence" refers to how clearly the evidence shows that recovering public chain data was a necessary step in the exploit.

| Tier | Case Name | Date | Chain / Network | Project Type | Real Financial Loss | Estimated Loss | Core On-Chain Sensitive Data Case | What the Attacker Read | How the Data Was Exposed | How That Information Became Profit | Primary Evidence Sources | Confidence |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| B | SmartBillions (predictable randomness) | 2017-10-05 (multi-source reporting) | Ethereum mainnet | Lottery / gambling | Yes | About 400 ETH (reported figure) | No, boundary: randomness / chain properties | The predictability that future `blockhash` becomes 0, plus visible state such as `block.number` | Exploitation of EVM `blockhash` behavior for blocks older than 256, together with public chain variables | The attacker waited for the correct window, making the outcome predictable and then extracting value from the pool; afterward the project address showed large outflows | Public discussion of the amount, the project's Hackathon article with contract address, and block explorer records showing a 1,100 ETH-scale outflow consistent with the project withdrawing remaining funds | Medium |
| B | FoMo3D (predicted randomness plus anti-contract restriction bypass) | 2018 (commonly cited in papers and postmortems) | Ethereum mainnet | Gambling / game | Public evidence of profit or unfair advantage exists, but loss figures differ | Estimates vary widely depending on source | No, boundary: weak randomness / chain properties plus adversarial strategy | Public PRNG inputs such as block attributes and `msg.sender` | Public block fields and observable transaction behavior | The attacker used contracts and timing to participate only when the odds were favorable, improving expected profit | Research articles with root-cause discussion, security-team analysis of suspicious winner-related transactions and miner incentives, and academic discussion of the incident | Medium |
| A (teaching representative) | Ethernaut Level "Vault" (read storage slot to recover `password`) | Teaching challenge | Ethereum test / educational environment | CTF / teaching | No | — | Yes | The slot containing `bytes32 private password` | Contract storage is publicly readable off-chain | Read the slot, recover the password, call `unlock(password)` | Writeups showing that the password sits in slot 1 and can be read via `getStorageAt`, plus SWC-136 as the general principle | High |
| A (teaching representative) | Chain Heist CTF "Account Unlock" (read slot to recover plaintext password) | 2019 | Ropsten at the time | CTF / teaching | No | — | Yes | The plaintext password stored in slot 0 | The contract stored `string public password` in storage | Read the slot, decode the string, pass the verification function | A writeup providing the contract address, storage-read command, and ASCII conversion process, plus secondary material placing it in the Chain Heist challenge set | High |
| A (teaching / standard sample) | SWC-136 OddEven (first player's move exposed in calldata) | Standard teaching sample | EVM generic | Game / small betting pattern | Depends on deployment; sample itself is not an incident disclosure | — | Yes | The first player's submitted number in transaction input | The transaction input is public, letting the second player observe it before choosing a move | The second player chooses a number that forces the parity outcome in their favor | The SWC-136 sample and explanation | Medium |
| B (teaching / boundary) | Capture the Ether "Guess the Secret Number" (hashed answer can be brute-forced) | Longstanding teaching challenge | Ethereum teaching environment | CTF / teaching | No | — | No, boundary: low entropy plus hash brute force | `answerHash` plus a tiny search space | The hash is embedded directly in code or state and can be brute-forced offline | Offline brute force reveals the answer, then the attacker calls `guess` | Teaching articles and writeups describing the attack path | High |
| C (excluded example) | Private-key leak / repository leak cases, such as a project committing keys to GitHub | Multiple incidents | Multiple chains | Operational security / key management | Yes | Can be large | No | Private keys or seed phrases | The secret was leaked externally, not on-chain | Direct account takeover | Incident reporting showing that the root cause was external credential exposure, not blockchain storage disclosure | High |

> The table intentionally places most large real-world losses in category B rather than category A. Based on public evidence available as of March 16, 2026, **strict A-class incidents, where a secret was simply read from on-chain data and that alone caused a major hack, are rare**. In contrast, B-class cases such as randomness, commit-reveal, and MEV-related mechanisms produce larger real-world consequences far more often, while A-class examples are concentrated in teaching systems, small games, and small-scale projects.

### Detailed Case Analysis

**Would the attack still work if public on-chain data were not readable?**

**SmartBillions (about 400 ETH): chain-property predictability made the winning logic readable** (Boundary case B)  
This incident is often cited as a classic example of why blockchain randomness is unsafe when derived from chain properties. The project used `blockhash`-related logic to determine gambling outcomes. Research and postmortem material explains that an attacker could simply wait more than 256 blocks, after which `blockhash` for the target block would return zero in the EVM, turning the outcome into something predictable. Public discussions state that the attacker first withdrew around 400 ETH and that the project then quickly withdrew the rest.  
From the on-chain side, the project's Hackathon article disclosed the jackpot contract address `0x5acE17...`. That address shows an approximately **1,100 ETH** outgoing transfer on **October 5, 2017**, which aligns with the narrative that the project operators withdrew the remaining balance. What remains less certain is the exact one-to-one mapping between the reported "400 ETH attacker withdrawal" and a complete set of attacker-side transactions, so the amount-to-transaction correspondence should be treated conservatively.

- **What was the sensitive information?** Strictly speaking, not a hidden password, but rather the predictability of the randomness rule itself, specifically the fact that `blockhash` becomes zero outside the 256-block window.  
- **How did the attacker obtain it?** Through public consensus rules plus visible block height and timing.  
- **What did the project misunderstand?** It treated chain attributes as unpredictable randomness, or failed to account for the `blockhash` availability window.  
- **How should it be fixed?** Use a safer randomness design such as a sound commit-reveal or an external verifiable randomness source, and never depend on predictable chain attributes.  
- **Key answer**: if chain data were not publicly readable or block properties were not observable, this class of prediction would become much harder. On a public blockchain, however, the exploit is exactly what happens when public information is mistaken for hidden entropy.

**FoMo3D: public block fields plus observable transaction behavior created controllable odds** (Boundary case B)  
FoMo3D is controversial because it is not a simple "read the password from storage" bug. Instead, it mixed **block properties such as timestamp, difficulty, miner address, and participant addresses** into a PRNG-like mechanism. Researchers have pointed out that because these inputs are public, an adversary can reconstruct the same randomness computation off-chain and then choose to interact only when the odds are favorable. Contract-level tricks, including bypassing weak "EOA only" assumptions during construction, can further amplify the edge.  
Security-team analysis of unusual winner-related blocks and high-fee transaction behavior also shows how, in a high-stakes game, attackers can exploit publicly visible chain mechanics and miner incentives.

- **What was the sensitive information?** Not a traditional secret, but rather values the project wrongly treated as secure randomness inputs.  
- **How did the attacker obtain them?** By observing public block fields and transaction behavior.  
- **Why is it easy to misclassify as A-class?** Because it still demonstrates that public chains do not keep secrets, yet it lacks the most explicit A-class feature: a password, salt, or similar value intended to remain secret being directly recovered.  
- **How should it be fixed?** Avoid chain-property randomness, use verifiable randomness or a correct commit-reveal design, and do not rely on checks such as code-length heuristics that contracts can bypass during construction.  
- **Key answer**: if the attacker could not observe block fields or transaction history, their ability to reproduce and time the attack would be severely limited. The exploit therefore depends heavily on public chain visibility.

**Ethernaut Vault and Chain Heist Account Unlock: standardized teaching evidence for the `private` / slot-obscurity misconception** (Teaching representatives of class A)  
These challenges reduce the problem to its purest form. The contract writes a password to storage, and the developer wrongly assumes that `private` or the lack of a getter function makes it confidential. In reality, anyone who understands storage layout can use `getStorageAt` or equivalent tooling to read the relevant slot. Chain Heist makes the point even more directly by showing a contract with a password string in storage slot 0 that can be decoded back into plaintext.

- **Key answer**: if contract state were truly unreadable from outside, these attacks would fail immediately. That is exactly why they serve as the clearest evidence that public chain readability alone can make the exploit possible.

## Why Strict Large-Loss Cases Are Rare and What Gets Misclassified

**Why strict A-class large real-world hacks are relatively uncommon**  
1) **The consensus-layer reality that "there are no secrets on-chain" is already well understood**. Solidity security guidance and SWC-136 have warned against storing private material in contract code or state for years, making this a low-cost, high-confidence audit finding in mature workflows.  
2) **Large projects rarely rely on passwords or answers for critical authorization**. In DeFi, control over valuable assets is usually gated by stronger primitives such as access control, signatures, governance, multisigs, or MPC, rather than a secret directly written on-chain.  
3) **Large real incidents often involve mixed causes**. Randomness, commit-reveal, reusable signatures, MEV, access control flaws, and application logic issues often overlap, making it difficult to isolate a pure "public storage disclosure" root cause.

**Well-known incidents that are often misclassified into this topic but should be excluded**  
- **Project private-key leaks**: these look like "secret exposure" but the secret came from GitHub or an operational environment, not public blockchain state.  
- **Frontend or domain hijacking**: the chain is only the settlement layer; the real root cause is compromise of the Web2 entry point or social engineering.  
- **The DAO and similar classic exploits**: these belong to reentrancy and business logic failure, not "recovering a secret from on-chain data."  
- **Ordinary AMM front-running or sandwiching**: these are transaction ordering and MEV problems. They only overlap with this topic when the copied material is itself a revealed secret or reusable credential.

## Risk Patterns and Defensive Guidance

### Common Risk Patterns

- **Most common data types involved**: passwords, answers, low-entropy secrets, game moves, reusable signature material, and randomness "seeds" that are actually public chain attributes.  
- **Most common mistaken beliefs**:  
  1) "`private` means secret";  
  2) "storage slots are hard to compute, so that is enough";  
  3) "unverified source code gives security through obscurity";  
  4) "block properties are good enough as randomness."  
- **Most common affected scenarios**: lotteries, competitions, gambling games, gatekeeping or authentication systems, claim mechanisms, and any design requiring fairness or unpredictability.  
- **Most common attacker benefit models**: guaranteed-win arbitrage, selective participation after learning the answer, reusing credentials to bypass identity checks in boundary cases, and breaking the integrity of the mechanism for economic or reputational gain.

### Defensive Guidance by Layer

**Design layer: accept that public blockchains do not have secrets**  
- Build the threat model correctly: on a public chain, **storage, calldata, events, and transaction history are all public**. If data must remain secret, keep it off-chain unless you are deliberately using a well-validated privacy system such as a suitable ZK or TEE-based design.  
- If passwords, answers, winning numbers, seeds, or salts must remain secret, only two broad options are sane:  
  1) keep them **off-chain**; or  
  2) store only **ciphertext on-chain**, with the decryption key never appearing in a publicly readable on-chain surface. Key rotation and operational usability must still be evaluated carefully.  
- For lotteries and games, prefer verifiable randomness or a properly engineered commit-reveal design. Do not treat `blockhash`, `timestamp`, `prevrandao`, or similar values as secure randomness.

**Contract layer: coding rules and auditable implementations**  
- Treat `private` correctly: it limits contract-level access but does not encrypt anything. Do not store sensitive strings, passwords, or key material in plaintext or low-entropy form in state.  
- For commit-reveal schemes:  
  - the commitment must include enough entropy, usually through a strong random salt;  
  - the reveal must bind identity or entitlement, such as `msg.sender` or a specific beneficiary, so that a copied reveal transaction cannot simply be replayed by someone else.  
- For credential-style authorization such as signatures, passwords, or tickets: do not reuse them as if they were static secrets; add nonces and domain separation, for example using EIP-712-style discipline, to reduce replay risk.

**Protocol layer: account for public visibility and MEV realities**  
- If the protocol requires reveal steps or sensitive parameters to be submitted through the public mempool, the design should assume searchers will observe and copy them. Bind benefits to a recipient, introduce timing protections, or use private transaction pathways where appropriate.  
- In cross-chain or bridge systems using hashlocks or HTLC-like patterns, ensure that revealing the preimage does not allow an arbitrary third party to claim funds. The claim path must strongly bind the beneficiary.

**Engineering process layer: audit, test, and automate checks**  
- **Audit checklist**:  
  1) search for keywords such as `password`, `secret`, `salt`, `seed`, `answer`, and `key`;  
  2) inspect state variables of types like `string`, `bytes`, and `bytes32`, and check whether constructors or initializers write externally supplied sensitive material to storage;  
  3) inspect lotteries and game contracts for state machines that write answers or winning values before participation closes.  
- **State replay testing**: use a fork plus `eth_getStorageAt` and debugger tooling to enumerate slots and verify whether a "read it and win" condition exists, especially for hidden fields inside mappings, structs, and dynamic arrays.  
- **Standards alignment**: incorporate "no secrets on-chain" and least-privilege principles into the security baseline and review checklists.

## Appendix and Deliverables

### Summary

1) **Core conclusion**: there are no secrets on a public blockchain. `private` is not encryption, and anything written to storage, calldata, or events can be read or inferred off-chain.  
2) **Most typical misconceptions**: treating `private`, slot obscurity, or unverified source code as confidentiality; treating block properties as randomness; or treating reveal data and signatures as one-time credentials without nonces or identity binding.  
3) **Observed real-world distribution**: strict A-class cases are relatively rare in major projects and appear more often in teaching material and small games; boundary cases involving randomness, commit-reveal, and MEV create larger economic consequences in practice, even though the shared root mistake is still misuse of public on-chain information.  
4) **Practical audit checklist**: search for dangerous secret-bearing keywords, derive storage layout, and verify with `eth_getStorageAt` in a forked environment whether readable state is directly exploitable.

### Ten Most Citable Cases

1) SmartBillions (about 400 ETH, boundary: randomness / chain properties)  
2) FoMo3D (boundary: weak randomness and observable chain properties plus adversarial strategy)  
3) Ethernaut "Vault" (teaching: recover `password` by reading storage)  
4) Chain Heist "Account Unlock" (teaching: plaintext password written to storage)  
5) SWC-136 OddEven (sample: the first player's input becomes public and lets the second player force a win)  
6) Capture the Ether "Guess the Secret Number" (teaching: low-entropy hashed answer can be brute-forced)  
7) Solidity's official "Private Information and Randomness" guidance  
8) Solidity's official storage layout documentation  
9) `eth_getStorageAt` as the standard interface proving that storage is routinely readable off-chain  
10) Representative private-key leak incidents used to clarify what should be excluded from this category

### Fifteen Most Authoritative Sources

1) SWC-136: Unencrypted Private Data On-Chain  
2) Solidity security considerations: Private Information and Randomness  
3) Solidity documentation: Layout of State Variables in Storage  
4) Ethereum.org JSON-RPC API documentation  
5) Ethereum Execution APIs JSON-RPC specification index  
6) MetaMask documentation for `eth_getStorageAt`  
7) EEA EthTrust Security Levels  
8) SmartBillions project Hackathon article disclosing the contract address  
9) Etherscan contract pages and transaction records  
10) SECBIT Labs analysis of suspicious FoMo3D winner-related transactions  
11) Academic papers discussing FoMo3D as a real attack example  
12) Raz0r / HITB material on predictable randomness  
13) High-quality Chinese translated Solidity security references  
14) Chinese smart-contract Top 10 security summaries  
15) Ethereum security education resources clarifying frequently misunderstood loss causes

### Cases With Remaining Disputes or Incomplete Evidence

- **SmartBillions**: public discussion and technical explanation are strong, but mapping the reported "400 ETH" exactly to specific attacker-side transactions still requires more granular internal transfer and call-level reconstruction.  
- **FoMo3D**: different papers and articles report different profit or loss scales. The strong conclusion is about the exploit mechanism, not the exact amount.

### Blind Spots and Limitations

- **Structural bias in incident databases**: public incident archives usually index cases under labels like reentrancy, access control, randomness, or oracle manipulation. As a result, strict A-class cases can be hidden inside broader categories such as game logic, randomness, or MEV.  
- **Long-tail small projects are hard to capture systematically**: many "password written on-chain" mistakes occur in tiny or short-lived projects that never receive a formal public postmortem.  
- **Language and platform scope**: this report is centered on EVM and Solidity. Other chains also expose public state, but the tooling and vulnerability patterns differ enough that they require a separate search surface.  
- **Limited per-transaction forensic reconstruction**: some older incidents, especially SmartBillions, would benefit from deeper internal-transaction tracing, flow clustering, and method-level reconstruction before amount-to-transaction linkage can be treated as fully complete.

### Out of Scope by Time

Events occurring after **March 16, 2026** are intentionally excluded from the main body of this report. Since the follow-up search was not extended beyond that date, no later incidents are listed here.
