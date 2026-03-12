# Research on Real-World ABI-Smuggling-Related DeFi / Smart Contract Security Incidents as of 2026-03-12

## Scope and Selection Criteria

This report includes only **security incidents that had actually occurred on mainnet / L2 / sidechains as of 2026-03-12 (Tokyo time), and that resulted in verifiable asset loss or fund movement**. It deliberately excludes CTFs, demos, purely theoretical PoCs, and "conceptual vulnerabilities" that were never exploited on-chain or whose transactions cannot be verified. `cite: turn8view0, turn13view0, turn24view0, turn11view0`

The case selection centers on the root-cause family you specified:  
(a) the selector / critical field read by the authorization check differs from the selector / critical field actually used by the execution path;  
(b) manual calldata parsing (`calldataload` / fixed offsets) produces **two parsing semantics** when combined with ABI dynamic offsets, nested bytes, multicall / forwarder / proxy / fallback / generic executor patterns;  
(c) the result is privilege bypass, arbitrary call, meta-transaction sender spoofing, multicall selector / parameter confusion, and similar issues. `cite: turn13view0, turn24view0, turn8view0, turn26view0`

Source priority follows: incident postmortems by the affected party / deep analysis by reputable security teams / audits or official disclosures / block explorer transactions and verified source code. This report primarily cites `entity: ["organization","BlockSec","web3 security firm"]`, `entity: ["company","OpenZeppelin","smart contract security"]`, `entity: ["company","CertiK","blockchain security firm"]`, `entity: ["organization","SlowMist","blockchain security firm"]`, `entity: ["organization","Decurity","web3 security firm"]`, and block explorer records from `entity: ["organization","Etherscan","ethereum block explorer"]` and `entity: ["organization","PolygonScan","polygon block explorer"]`. `cite: turn8view0, turn6view0, turn26view0, turn24view0, turn21view0, turn9view0`

## Vulnerability Family Definition and Similarity Ranking

### ABI Smuggling in the Strict Sense

This report limits "strict ABI smuggling" to vulnerabilities where **the same calldata is interpreted by two pieces of logic under two different ABI semantics**:

1. The **authorization check** (Auth) reads a selector or key field from some calldata position, typically via a hard-coded offset or an assumed fixed layout.  
2. The **execution path** (Exec) later parses the same parameter under a different rule set, typically by decoding according to the actual ABI offset, via `abi.decode`, or via another assembly routine that computes offsets differently.  
3. The attacker crafts an encoding that is ABI-valid but non-canonical, especially by exploiting the freedom of dynamic parameter offsets, so that **the field seen by Auth != the field actually used by Exec**, enabling privilege bypass or spoofing of payer / sender / selector. `cite: turn13view0, turn22view1, turn21view0`

> The key point is that this is not undefined behavior from arbitrary garbage bytes. It exploits the ABI's legitimate freedom around dynamic offsets to create two different parsing results. `cite: turn13view0, turn22view1`

### The Same Family Under Different Names

If an incident is not explicitly labeled "ABI smuggling" but matches any of the following traits, this report classifies it into the same family while assigning it a lower similarity score:

- During **calldata / memory reconstruction**, a controllable length / offset / pointer arithmetic bug, such as underflow, causes a suffix or execution context to be overwritten or forged, so the authorization check runs against the wrong context. `cite: turn24view0, turn25view0`  
- In a **meta-transaction forwarder + multicall + delegatecall / self-call** combination, the assumption that `_msgSender()` or the context suffix can recover the sender from the end of calldata breaks, so the authenticated `from` differs from the address actually treated as sender. `cite: turn8view0, turn6view0, turn26view0`  
- A **callback / generic executor** treats an external callback as a trusted execution source, but caller validation or context binding fails, allowing an arbitrary external party to trigger transfers of assets previously approved by the victim. This is closer to "arbitrary callback trigger + passive spending of approved assets" and is less similar to ABI smuggling's parsing divergence. `cite: turn11view0, turn12search9`  

### Similarity Ranking Method

Cases are ranked by how structurally isomorphic they are to strict ABI smuggling, in the following order:

1. **Auth reads a fixed or wrong offset, while Exec decodes using the ABI offset**. This is the most canonical pattern. `cite: turn13view0, turn22view1`  
2. **Hand-written calldata / memory reconstruction** allows later parsed fields to be forged or overwritten, decoupling the permission check from the execution context. `cite: turn24view0, turn25view0`  
3. **Meta-tx / multicall context parsing errors** lead to sender spoofing. At bottom, this is still "authenticated sender != actual execution sender." `cite: turn8view0, turn26view0, turn6view0`  
4. Callback caller checks / trust-boundary issues. This is the least similar to ABI parsing divergence, but it often co-occurs with low-level bytes handling in routers and aggregators. `cite: turn11view0`  

## Case List (Ordered from Most to Least Similar to ABI Smuggling)

### Case 1: V4 Router by z0r0z (Strict ABI Smuggling: Dynamic Bytes Offset Induces "Authenticated Payer != Actual Payer")

**Protocol / Component**: A third-party Router based on `entity: ["company","Uniswap","dex protocol"]` v4 Router (deployer: `entity: ["people","z0r0z","ethereum developer"]`). `cite: turn13view0, turn15view0, turn30search3`  
**Incident date**: 2026-03-03 06:17:11 UTC (on-chain timestamp). `cite: turn21view0`  
**Loss**: 42,606.959179 USDC according to Etherscan outflow records; the attacker received 21.197984596759249607 ETH from the swap. `cite: turn21view0`  

**Exploited function / module**: The inline assembly authorization check inside Router `swap(bytes calldata data, uint256 deadline)`. `cite: turn22view1, turn13view0`  

**Root cause (strict ABI smuggling)**:  
The Router tried to implement the equivalent of `require(abi.decode(data,(BaseData)).payer == msg.sender)`, but used a hard-coded gas-optimized read: comparing `calldataload(164)` against `caller()`. This implicitly assumes the ABI offset of `bytes data` is always `0x40`, so the payer field always sits at absolute calldata byte 164 (`0xA4`). The ABI spec, however, does **not guarantee** that a dynamic parameter offset must be the canonical one. An attacker can craft an ABI-valid but non-standard encoding, move the `bytes data` offset to `0xc0`, place the attacker's address at byte 164 so Auth passes, while leaving the real payer field at the end of `data` as the victim's address. The Exec path then uses the victim as payer and drains already approved assets. `cite: turn13view0, turn22view1, turn21view0`  

**Attack path (verifiable on-chain)**:  
The attacker selected a victim address that had previously approved the Router and built non-canonical calldata:  
- Auth stage: `calldataload(164)` reads the attacker's address, so the `Unauthorized()` check passes. `cite: turn22view1, turn13view0`  
- Exec stage: downstream logic such as `_unlockAndDecode(data)` decodes `data` using the new bytes offset, yielding "victim payer + attacker receiver". The Router pays 42,606.959179 USDC from the victim to the Uniswap v4 Pool Manager, and the attacker receives the resulting ETH. `cite: turn21view0, turn13view0`  

**Attack transaction or addresses**:  
- Attack tx: `0xfe34c4beee447de536bbd3d613aa0e3aa7eeb63832e9453e4ef3999924ab466a` (Etherscan shows USDC flowing from the victim to the Pool Manager and ETH from the Pool Manager to the attacker). `cite: turn21view0`  
- Attacker: `0xd6B7e831D64e573278f091AA7E68Fbf2A8FA9916`. `cite: turn21view0, turn23view0`  
- Victim (spoofed payer): `0x65A8F07Bd9A8598E1b5B6C0a88F4779DBC077675` (labeled by Etherscan as an EIP-7702 Delegated address). `cite: turn21view0, turn16view1`  
- Vulnerable Router: `0x00000000000044a361Ae3cAc094c9D1b14Eece97` (verified source on Etherscan includes the `calldataload(164)` logic). `cite: turn22view1, turn15view0`  

**Primary references**: BlockSec incident write-up and root-cause explanation, plus Etherscan contract source and transaction records. `cite: turn13view0, turn22view1, turn21view0`  

---

### Case 2: 1inch Fusion V1 Third-Party Resolver (Same Family: Hand-Written Calldata / Memory Reconstruction Lets Resolver Identity and Permission Context Be Forged)

**Protocol / Component**: The `entity: ["company","1inch","dex aggregator"]` Fusion V1 Settlement contract (legacy / compatibility-retained version) and a third-party market maker / Resolver contract from `entity: ["organization","TrustedVolumes","market maker"]`. `cite: turn24view0, turn25view0`  
**Incident date**: 2025-03-05 (attack occurred around 17:00 UTC). `cite: turn24view0`  
**Loss**: More than USD 5,000,000; most funds were later returned, with the attacker retaining a bounty-like remainder. `cite: turn24view0, turn25view0`  

**Exploited function / module**: The Yul / assembly calldata copy-and-suffix-concatenation logic inside Settlement `_settleOrder(bytes calldata data, address resolver, ...)`. `cite: turn24view0, turn25view0`  

**Root cause (same family, not strict ABI-offset smuggling, but structurally similar)**:  
To execute orders, Settlement copies calldata into memory, patches certain length fields, and appends a dynamic suffix containing context such as `totalFee` and `resolver`. The core bug is that the suffix write position is computed as `ptr + interactionOffset + interactionLength`, where `interactionLength` is a **32-byte attacker-controlled value**. By setting it to `0xffff...fe00` (`-512`), the attacker triggers pointer underflow and causes the real suffix to be written into a padding area prepared in advance by the attacker. Subsequent parsing then treats the attacker's fake interaction structure as the suffix, ultimately **forging the resolver / execution context** and triggering arbitrary `resolveOrders` calls against the victim's resolver contract. `cite: turn24view0, turn25view0`  

**Attack path (key steps)**:  
1. Craft seemingly normal order data, but use padding plus an invalid `interactionLength` so the memory patch logic underflows. `cite: turn24view0`  
2. Use the forged suffix so the final execution path invokes `resolveOrders(...)` on the victim resolver, which trusted only `msg.sender == Settlement`, thereby allowing passive fund release / transfers. `cite: turn24view0, turn25view0`  

**Attack transactions or addresses (verifiable artifacts from Decurity)**:  
- Victim (`TrustedVolumes`) address: `0xb02f39e382c90160eb816de5e0e428ac771d77b5`. `cite: turn24view0`  
- Settlement V1 address: `0xa88800cd213da5ae406ce248380802bd53b47647`. `cite: turn24view0`  
- Key attack txs (excerpt):  
  - `0x62734ce80311e64630a009dd101a967ea0a9c012fabbfce8eac90f0f4ca090d6`  
  - `0x74bc4d5dc7f8da468788da6087bb9f73465966ab5b8cf9cf1053d98e78a9bf96` `cite: turn24view0`  
- Related attacker addresses: exploit deployer `0xa7264a43a57ca17012148c46adbc15a5f951766e`, exploit contract `0x019bfc71d43c3492926d4a9a6c781f36706970c9`, fund receiver `0xbbb587e59251d219a7a05ce989ec1969c01522c0`. `cite: turn24view0`  

**Primary references**: Decurity postmortem with full technical detail and verifiable addresses / transactions, plus BlockSec's annual-summary-level independent characterization of the root cause. `cite: turn24view0, turn25view0`  

---

### Case 3: TIME Token (Same Family: ERC-2771 Forwarder + Multicall + Delegatecall Causes Calldata Truncation and Sender Spoofing)

**Protocol / Component**: The affected token TIME, derived from a third-party contract-template ecosystem using the ERC-2771 + Multicall pattern. `cite: turn8view0, turn26view0, turn6view0`  
**Incident date**: 2023-12-07. `cite: turn6view0, turn8view0, turn26view0`  
**Loss**: Roughly 84.59 ETH based on OpenZeppelin's observed in-the-wild exploit; CertiK reports about 89.5 ETH in losses and about 84.6 ETH in attacker profit, roughly USD 188K. `cite: turn8view0, turn31search5, turn6view0`  

**Exploited function / module**:  
- Forwarder `execute()` (verifies the `req.from` signature, appends it to calldata, then `call`s the target). `cite: turn6view0, turn26view0`  
- Token contract `multicall(bytes[])` (uses `delegatecall` for each subcall). `cite: turn6view0, turn26view0`  
- ERC-2771 `_msgSender()` (extracts the original sender from the final 20 bytes of calldata). `cite: turn8view0, turn26view0`  

**Root cause (same family: authenticated `from` != parsed `_msgSender`)**:  
The Forwarder is intended to append the signed `req.from` as a calldata suffix so the target contract can recover the real sender using ERC-2771 semantics. In practice, the Forwarder does `abi.encodePacked(req.data, req.from)`. But when `req.data` itself calls `multicall(bytes[])`, ABI decoding slices sub-elements according to the offsets and lengths of `bytes[]`, which can cause the appended `req.from` to be **truncated out of the expected position** by multicall parsing. Multicall then executes subcalls such as `burn()` via `delegatecall`, keeping `msg.sender` equal to the forwarder and satisfying `isTrustedForwarder`, while `_msgSender()` reads the last 20 bytes of the *actual* calldata and recovers an attacker-controlled address, such as an AMM pool. Sensitive burn / transfer logic therefore executes on behalf of the wrong identity. `cite: turn6view0, turn8view0, turn26view0`  

**Attack path (TIME case, verifiable)**:  
1. Acquire TIME tokens and prepare the target pool. `cite: turn6view0, turn26view0`  
2. Use `Forwarder.execute` to call token `multicall`; multicall uses `delegatecall` to execute `burn()`, directly burning a large amount of TIME from the pool and distorting price. `cite: turn6view0, turn26view0`  
3. Arbitrage out WETH / ETH against the manipulated price. `cite: turn6view0, turn26view0`  

**Attack transaction or addresses**:  
- Attack tx: `0xecdd111a60debfadc6533de30fb7f55dc5ceed01dfadd30e4a7ebdb416d2f6b6`. `cite: turn6view0, turn26view0, turn8view0`  
- Attacker address: `0xfde0d1575ed8e06fbf36256bcdfa1f359281455a`; attacker contract: `0x6980a47bee930a4584b09ee79ebe46484fbdbdd0`. `cite: turn26view0, turn6view2`  

**Primary references**: OpenZeppelin public disclosure with real in-the-wild transactions and amounts, CertiK incident analysis, and SlowMist's detailed attack walkthrough with transactions and addresses. `cite: turn8view0, turn6view0, turn26view0`  

---

### Case 4: Swopple (Swop) / Polygon ERC2771+Multicall In-The-Wild Exploit (Same Family: Sender Spoofing, Smaller Loss, Canonical Structure)

**Protocol / Component**: Swopple (Swop)-related contracts / pools on Polygon. The attack pattern is the same family as the ERC-2771 + Multicall address spoofing disclosed by OpenZeppelin. `cite: turn8view0, turn9view0`  
**Incident date**: 2023-12-07 08:48:55 UTC (PolygonScan record). `cite: turn9view0`  
**Loss**: About 17.3K USDC.e. PolygonScan shows 17,377.381133 USDC.e flowing from the Uniswap v3 pool to the attacker; OpenZeppelin gives roughly 17,394 USDC. `cite: turn9view0, turn8view0`  

**Exploited function / module**: Same vulnerability family as Case 3. ERC-2771 recovers sender from the calldata suffix, and multicall `delegatecall` lets the subcall sender be spoofed. PolygonScan directly labels this transaction "ERC2771 Exploiter." `cite: turn9view0, turn8view0`  

**Root cause (same family)**:  
Again, the trusted forwarder / context-suffix assumption is broken by multicall's ABI slicing and delegatecall semantics, so `_msgSender()` or similar logic reads an attacker-controlled address and uses it to execute unintended mint / burn / transfer paths against the pool or token. `cite: turn8view0, turn9view0`  

**Attack path (verifiable from transfers)**:  
The transaction shows a massive mint of Swop tokens from the null address, followed by roughly 17,377 USDC.e flowing from the Uniswap v3 USDC / Swop pool to attacker-related addresses. `cite: turn9view0`  

**Attack transaction or addresses**:  
- Attack tx on Polygon: `0x1b0e27f10542996ab2046bc5fb47297bcb1915df5ca79d7f81ccacc83e5fe5e4`. `cite: turn9view0, turn8view0`  
- Attacker (as labeled by PolygonScan): `0x0a4311b6a2E6DBC5b6A1a0C2bD77B3D83F220a1C`. `cite: turn9view0`  

**Primary references**: OpenZeppelin public disclosure listing the Polygon transaction and amount, plus the PolygonScan transaction details page for transfer and timestamp verification. `cite: turn8view0, turn9view0`  

---

### Case 5: ParaSwap Augustus V6 (Lower-Similarity Same Family: Missing Callback Caller Validation Enables a Forged Execution Source, an Arbitrary Call Auth Bypass Pattern)

**Protocol / Component**: `entity: ["organization","ParaSwap","dex aggregator"]` Augustus V6, with one Ethereum-side contract identifier being `0x00000000FdAC7708D0D360BDDc1bc7d097F47439`. `cite: turn11view0, turn12search8`  
**Incident date**: The issue was confirmed and publicly handled on 2024-03-20. Initial abnormal transactions occurred during the 2024-03-18 to 2024-03-20 window, followed by additional attacks that reused old approvals. `cite: turn11view0`  
**Loss**: The postmortem summarizes roughly USD 24,000 from the initial exploit, roughly USD 1,100,000 from follow-up attacks, roughly USD 3,400,000 rescued by whitehats, with some refunds and DAO coverage. `cite: turn11view0`  

**Exploited function / module**: `uniswapV3SwapCallback()`, the callback logic used in the gas-optimized direct Uniswap V3 swap path. `cite: turn11view0`  

**Root cause (relationship to ABI smuggling)**:  
This was not a typical ABI-offset smuggling incident. It is closer to an arbitrary-call auth bypass caused by a failed callback trust boundary: `uniswapV3SwapCallback()` is supposed to be callable only by a real Uniswap V3 pool, but in some branches the caller check was missing or not correctly enforced. An attacker could therefore create a fake pool and trigger the callback, causing the contract to transfer or pay out user-approved assets inside the callback. Its similarity to the ABI-smuggling family lies in the shared structure of **forged execution source / forged execution context leading to passive spending of approved assets**, but the root cause here is caller identity binding failure rather than divergence caused by ABI dynamic offsets. `cite: turn11view0`  

**Attack path (per postmortem)**:  
The attacker invoked the callback from a forged pool and used users' existing approvals to make Augustus V6 spend user tokens inside the callback, affecting multiple networks integrated with Uniswap V3. `cite: turn11view0`  

**Attack transaction or addresses (verifiable leads)**:  
- Contract address on Ethereum: `0x00000000FdAC7708D0D360BDDc1bc7d097F47439`. `cite: turn12search8`  
- Whitehat fund collection address (Safe Wallet): `0x66E90d840D7C4F3473E25dD8ca361747058c6Db0`, used for fund aggregation and refund tracking. `cite: turn11view0`  

**Primary references**: Velora (ParaSwap) official postmortem. `cite: turn11view0`  

## Root-Cause Mapping and Summary of "ABI Smuggling Similarity"

Using the question "does the field used by authorization / context binding match the field used by the actual execution path?", the incidents above map to your root-cause list as follows:

- **Case 1 (V4 Router by z0r0z)**: a canonical "dynamic bytes offset + fixed-offset `calldataload`" mismatch, where Auth and Exec disagree on the same field (payer). It sits at the intersection of "manual calldata parsing with fixed offsets", "ABI dynamic parameter offset causing privilege bypass", and "generic router parsing errors", and fully satisfies the definition of strict ABI smuggling. `cite: turn13view0, turn22view1, turn21view0`  
- **Case 2 (1inch Fusion V1)**: hand-written calldata / memory reconstruction plus an attacker-controlled length field causes pointer underflow, forges the resolver context, and results in an arbitrary call auth bypass. It belongs to the same family as selector / offset confusion, but does not rely on the validity of ABI dynamic offsets. It relies on assembly pointer arithmetic failure. `cite: turn24view0, turn25view0`  
- **Cases 3 and 4 (TIME and Swopple)**: meta-transaction forwarder + multicall / delegatecall + context-suffix parsing errors. At core, the authenticated source (the `from` verified by the forwarder) differs from the execution source (the value parsed by token `_msgSender()`). This matches your category of calldata parsing errors in meta-transaction forwarders, multicall, proxy / fallback / generic executor flows. `cite: turn8view0, turn6view0, turn26view0, turn9view0`  
- **Case 5 (ParaSwap)**: closer to missing callback caller checks that let arbitrary external callers trigger payment callbacks. Its similarity to ABI smuggling lies mainly in the forged execution context leading to passive payment of approved assets, but its core cause is not ABI offset / selector divergence, so it ranks lowest. `cite: turn11view0`  

## Actionable Review and Defense Checklist

The following checks directly cover the major patterns in this report under the common theme of "preventing two parsing semantics":

First, whenever `bytes calldata` or `bytes memory` participates in authorization decisions, force Auth and Exec to use the **same parsed result**. In Case 1, Auth used a hard-coded `calldataload(164)` while Exec decoded according to the ABI offset, creating an attacker-controllable divergence. During audit, any pattern that reads dynamic-parameter content through a **fixed absolute offset** should be treated as high risk. `cite: turn22view1, turn13view0`  

Second, any flow that copies calldata into memory, patches it, appends data, and then performs an external call, like Case 2, must treat all length and offset fields as untrusted input. That requires explicit bounds checks, prevention of arithmetic underflow / overflow, and guarantees that write positions cannot move backward and overwrite earlier fields, especially security-critical suffix / context fields. `cite: turn24view0, turn25view0`  

Third, any contract combining ERC-2771, or any variant that carries sender info at the tail of calldata, with multicall / delegatecall / self-call must guarantee:  
- suffix-length recognition and trimming in `_msgData()` / `_msgSender()` stay consistent across nested call layers and cannot be bypassed by subcalls;  
- multicall subcalls must not allow the attacker to control the final 20 bytes and turn them into a new sender.  
OpenZeppelin's public disclosure shows that this pattern was exploited for real money more than once. `cite: turn8view0, turn26view0, turn6view0`  

Fourth, in router / aggregator / callback designs, every callback that can trigger payment or token transfer must strongly bind caller identity to execution context, for example via strict validation of pool address, factory address, initcode hash, and callback-parameter consistency. Case 5 shows that even without ABI-offset divergence, missing caller validation in one branch is enough to create a forged execution source and drain approved assets. `cite: turn11view0`  

## Appendix: Source Links (Key Evidence for Each Case)

All links below are primary sources, including postmortems, official disclosures, block explorers, and verified source code, so each case can be independently checked.

### Case 1: V4 Router by z0r0z (Strict ABI Smuggling)
```text
BlockSec weekly incident roundup (includes root cause and attack tx link):
https://blocksec.com/blog/weekly-web3-security-incident-roundup-mar-2-mar-8-2026

Etherscan attack tx:
https://etherscan.io/tx/0xfe34c4beee447de536bbd3d613aa0e3aa7eeb63832e9453e4ef3999924ab466a

Etherscan vulnerable Router source (includes calldataload(164)):
https://etherscan.io/address/0x00000000000044a361ae3cac094c9d1b14eece97

Etherscan victim address (spoofed payer):
https://etherscan.io/address/0x65a8f07bd9a8598e1b5b6c0a88f4779dbc077675

Source repository (Router project):
https://github.com/z0r0z/v4-router
```

### Case 2: 1inch Fusion V1 Resolver (Calldata Corruption / Suffix Injection)
```text
Decurity postmortem (includes root cause, victim / attacker addresses, and attack tx list):
https://blog.decurity.io/yul-calldata-corruption-1inch-postmortem-a7ea7a53bfd9

BlockSec 2025 annual incident summary mention:
https://blocksec.com/blog/top-10-awesome-security-incidents-in-2025
```

### Case 3: TIME Token (ERC2771 + Multicall Sender Spoofing)
```text
OpenZeppelin public disclosure (includes in-the-wild exploit tx and amount):
https://www.openzeppelin.com/news/arbitrary-address-spoofing-vulnerability-erc2771context-multicall-public-disclosure

CertiK incident analysis (TIME Token exploit):
https://www.certik.com/blog/time

SlowMist deep dive (includes attack steps, tx, and addresses):
https://slowmist.medium.com/an-in-depth-analysis-of-arbitrary-address-spoofing-attacks-5e6df80d5dae

Etherscan attack tx:
https://etherscan.io/tx/0xecdd111a60debfadc6533de30fb7f55dc5ceed01dfadd30e4a7ebdb416d2f6b6
```

### Case 4: Swopple / Polygon (One of the ERC2771+Multicall In-The-Wild Exploits)
```text
OpenZeppelin public disclosure (lists the Polygon attack tx):
https://www.openzeppelin.com/news/arbitrary-address-spoofing-vulnerability-erc2771context-multicall-public-disclosure

PolygonScan attack tx:
https://polygonscan.com/tx/0x1b0e27f10542996ab2046bc5fb47297bcb1915df5ca79d7f81ccacc83e5fe5e4
```

### Case 5: ParaSwap Augustus V6 (Missing Callback Caller Validation / Fake Pool)
```text
Velora (ParaSwap) official postmortem:
https://veloradex.medium.com/post-mortem-augustus-v6-vulnerability-of-march-20th-2024-5df663a4bf01

Etherscan: one identified Augustus V6 contract address:
https://etherscan.io/address/0x00000000fdac7708d0d360bddc1bc7d097f47439
```
