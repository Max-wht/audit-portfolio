# 截至 2026-03-12 已真实发生的 ABI-smuggling 同族 DeFi/智能合约安全事件研究

## 研究范围与筛选标准

本报告仅收录**截至 2026-03-12（东京时间）在主网/二层/侧链上真实发生、并造成可核验资产损失或资金转移的安全事件**，并刻意排除：CTF、demo、纯理论 PoC、未上链或无法核验交易的“概念漏洞”。`cite: turn8view0, turn13view0, turn24view0, turn11view0`

事件筛选以你给出的根因家族为中心：  
（a）授权检查读取的 selector/关键字段 与 实际执行路径使用的 selector/关键字段不一致；  
（b）手动解析 calldata（`calldataload` / 固定偏移）在遇到 ABI 动态参数 offset、嵌套 bytes、multicall/forwarder/proxy/fallback/generic executor 场景时产生“**两套解析语义**”；  
（c）因此出现权限绕过、任意调用（arbitrary call）、meta-transaction sender spoofing、multicall selector/参数错位等。`cite: turn13view0, turn24view0, turn8view0, turn26view0`

高质量/一手来源优先级遵循：事故方 postmortem / 权威安全团队深度分析 / 审计或官方披露 / 区块浏览器交易与合约源码（可验证）。本报告主要引用 `entity: ["organization","BlockSec","web3 security firm"]`、`entity: ["company","OpenZeppelin","smart contract security"]`、`entity: ["company","CertiK","blockchain security firm"]`、`entity: ["organization","SlowMist","blockchain security firm"]`、`entity: ["organization","Decurity","web3 security firm"]`、以及区块浏览器记录（`entity: ["organization","Etherscan","ethereum block explorer"]` 与 `entity: ["organization","PolygonScan","polygon block explorer"]`）。`cite: turn8view0, turn6view0, turn26view0, turn24view0, turn21view0, turn9view0`

## 漏洞家族定义与相似度排序方法

### 严格意义上的 ABI smuggling

本报告将“严格 ABI smuggling”限定为一种**“同一份 calldata 被两段逻辑以两套 ABI 语义解释”**的漏洞：

1) **授权检查**（Auth）从 calldata 的某个位置读取 selector/关键字段（常见：硬编码偏移或假设固定 layout）。  
2) **实际执行**（Exec）在后续路径中用另一套规则解析同一参数（常见：真正按 ABI offset 解码，或通过 `abi.decode` / 另一段 assembly 计算偏移）。  
3) 攻击者构造“ABI 合法但非 canonical”的编码（尤其是动态参数 offset 可合法指向任意对齐位置），使得 **Auth 看到的字段 ≠ Exec 实际使用的字段**，从而绕过权限或冒用 payer/sender/selector。`cite: turn13view0, turn22view1, turn21view0`

> 关键点：这不是“字节串随便乱填”的未定义行为，而是利用 ABI 允许的动态 offset 自由度，制造两套解析结果。`cite: turn13view0, turn22view1`

### “同族但不同名”的 ABI smuggling 家族

若事件不直接以“ABI smuggling”命名，但满足以下任一特征，本报告将其归入“同族”并在排序中降低相似度等级：

- **calldata/内存重构**阶段存在可控长度/偏移/指针运算错误（例如 underflow）导致“尾部字段（suffix）/上下文（context）被覆盖或伪造”，最终使权限检查基于错误上下文。`cite: turn24view0, turn25view0`  
- **meta-transaction forwarder + multicall + delegatecall/self-call** 组合下，`_msgSender()`/context suffix 从 calldata 末尾抽取地址的假设被破坏，造成“认证的 from 与实际被当作 sender 的地址不一致”。`cite: turn8view0, turn6view0, turn26view0`  
- **callback / generic executor** 把外部合约回调当作可信执行源，但 caller 校验/上下文绑定失败，导致“任意外部方触发受害者授权资产转移”。（这类更偏“任意回调触发 + 授权资产被动支付”，与 ABI smuggling 的“解析分歧”相似度较低。）`cite: turn11view0, turn12search9`  

### 相似度排序方法

按“与严格 ABI smuggling 的结构同构程度”排序，优先级依次为：

1) **Auth 读取固定偏移（或错误偏移） + Exec 按 ABI offset 解码**（最典型）。`cite: turn13view0, turn22view1`  
2) **手搓 calldata/内存重构**导致“后续解析字段被伪造/覆盖”，从而让权限检查与执行上下文脱钩。`cite: turn24view0, turn25view0`  
3) **meta-tx/multicall context 解析错误**导致 sender 伪装（address spoofing），本质也是“认证 sender ≠ 实际执行 sender”。`cite: turn8view0, turn26view0, turn6view0`  
4) callback caller check/边界信任问题（与 ABI 解析分歧同族性最低，但常与 bytes 参数、路由器/聚合器的低级优化代码共现）。`cite: turn11view0`  

## 案例清单（按 ABI-smuggling 相似度从高到低排序）

### 案例一：V4 Router by z0r0z（严格 ABI smuggling：动态 bytes offset 诱导“认证 payer ≠ 实际 payer”）

**协议/组件**：以 `entity: ["company","Uniswap","dex protocol"]` v4 Router 为基础的第三方 Router（部署者：`entity: ["people","z0r0z","ethereum developer"]`）。`cite: turn13view0, turn15view0, turn30search3`  
**事件日期**：2026-03-03 06:17:11 UTC（链上交易时间）。`cite: turn21view0`  
**损失金额**：42,606.959179 USDC（Etherscan 以 USDC 流出计）；攻击者接收 21.197984596759249607 ETH（池内换出）。`cite: turn21view0`  

**被利用函数/模块**：Router 合约 `swap(bytes calldata data, uint256 deadline)` 的内联汇编授权检查。`cite: turn22view1, turn13view0`  

**根因（严格 ABI smuggling）**：  
Router 试图等价实现 `require(abi.decode(data,(BaseData)).payer == msg.sender)`，但为省 gas 使用硬编码读取：`calldataload(164)` 与 `caller()` 比较。该实现隐含假设：`bytes data` 的 ABI offset 恒为 `0x40`，从而 payer 恒落在 calldata 的绝对字节位置 164（0xA4）。然而 ABI 规范并**不保证**动态参数 offset 必须为 canonical 值；攻击者可构造 ABI 合法但非标准的 encoding，将 `bytes data` 的 offset 改到 `0xc0`，在 164 字节处塞入攻击者地址通过 Auth，而真正 `data` 尾部中的 payer 字段仍写成受害者地址，Exec 路径使用受害者作为 payer 拉走其已授权资产。`cite: turn13view0, turn22view1, turn21view0`  

**攻击路径（可复核链上行为）**：  
攻击者挑选一个曾对该 Router 授权的受害者地址，构造非 canonical calldata：  
- Auth 阶段：`calldataload(164)` 读到攻击者地址 → 通过 `Unauthorized()` 检查。`cite: turn22view1, turn13view0`  
- Exec 阶段：`_unlockAndDecode(data)` 等下游逻辑按新的 bytes offset 解码 data，读取到“受害者 payer + 攻击者 receiver”，从受害者地址向 Uniswap v4 Pool Manager 支付 42,606.959179 USDC，并把兑换所得 ETH 给攻击者。`cite: turn21view0, turn13view0`  

**攻击交易或地址**：  
- 攻击交易：0xfe34c4beee447de536bbd3d613aa0e3aa7eeb63832e9453e4ef3999924ab466a（Etherscan 显示 USDC 从受害者流向 Pool Manager，ETH 从 Pool Manager 流向攻击者）。`cite: turn21view0`  
- 攻击者：0xd6B7e831D64e573278f091AA7E68Fbf2A8FA9916。`cite: turn21view0, turn23view0`  
- 受害者（被冒用 payer）：0x65A8F07Bd9A8598E1b5B6C0a88F4779DBC077675（Etherscan 标注为 EIP-7702 Delegated 地址）。`cite: turn21view0, turn16view1`  
- 存在漏洞的 Router：0x00000000000044a361Ae3cAc094c9D1b14Eece97（源码可在 Etherscan 验证到 `calldataload(164)` 逻辑）。`cite: turn22view1, turn15view0`  

**主要参考来源**：BlockSec 事件复盘与根因说明、Etherscan 合约源码与交易记录。`cite: turn13view0, turn22view1, turn21view0`  

---

### 案例二：1inch Fusion V1 第三方 Resolver（同族：手搓 calldata/内存重构导致 resolver 身份与权限上下文被“伪造植入”）

**协议/组件**：`entity: ["company","1inch","dex aggregator"]` 的 Fusion V1 Settlement 合约（旧版本/兼容性保留）与第三方做市商/Resolver `entity: ["organization","TrustedVolumes","market maker"]` 合约。`cite: turn24view0, turn25view0`  
**事件日期**：2025-03-05（约 17:00 UTC 附近发生攻击）。`cite: turn24view0`  
**损失金额**：超过 5,000,000 美元；事后多数资金返还，攻击者保留“bounty”式尾款。`cite: turn24view0, turn25view0`  

**被利用函数/模块**：Settlement 内部 `_settleOrder(bytes calldata data, address resolver, ...)` 的 Yul/assembly calldata 复制与 suffix 拼接逻辑。`cite: turn24view0, turn25view0`  

**根因（同族，非严格 ABI offset smuggling，但结构相近）**：  
Settlement 为执行订单会在内存中“复制 calldata + patch 某些长度字段 + 追加动态 suffix（包含 totalFee、resolver 等上下文）”。核心 bug 在于：suffix 写入位置使用 `ptr + interactionOffset + interactionLength` 计算，而 `interactionLength` 是**攻击者可控的 32 字节值**，可被设置为 `0xffff...fe00`（-512）触发指针下溢，使真正的 suffix 被写到“攻击者预先布置的 padding 区”，从而让后续解析把攻击者提供的“假交互结构”当作 suffix 使用，最终达到**伪造 resolver/上下文**、触发对受害者 resolver 合约的任意 `resolveOrders` 调用。`cite: turn24view0, turn25view0`  

**攻击路径（关键步骤）**：  
1) 构造看似正常的订单数据，并用 padding + 非法 `interactionLength` 引导 Settlement 在内存 patch 时发生 underflow。`cite: turn24view0`  
2) 通过伪造的 suffix，让 Settlement 在最终执行路径中调用受害者 resolver 的 `resolveOrders(...)`，而受害者 resolver 仅以 `msg.sender == Settlement` 作为信任边界，导致被动放款/转账。`cite: turn24view0, turn25view0`  

**攻击交易或地址（Decurity 给出可复核工件）**：  
- 受害者（TrustedVolumes）地址：0xb02f39e382c90160eb816de5e0e428ac771d77b5。`cite: turn24view0`  
- Settlement V1 地址：0xa88800cd213da5ae406ce248380802bd53b47647。`cite: turn24view0`  
- 关键攻击交易（列表节选）：  
  - 0x62734ce80311e64630a009dd101a967ea0a9c012fabbfce8eac90f0f4ca090d6  
  - 0x74bc4d5dc7f8da468788da6087bb9f73465966ab5b8cf9cf1053d98e78a9bf96 `cite: turn24view0`  
- 攻击相关地址：exploit deployer 0xa7264a43a57ca17012148c46adbc15a5f951766e、exploit contract 0x019bfc71d43c3492926d4a9a6c781f36706970c9、fund receiver 0xbbb587e59251d219a7a05ce989ec1969c01522c0。`cite: turn24view0`  

**主要参考来源**：Decurity 事故复盘（包含完整技术细节与可核验地址/交易）、BlockSec 年度事件总结对根因的独立概括。`cite: turn24view0, turn25view0`  

---

### 案例三：TIME Token（同族：ERC-2771 forwarder + multicall + delegatecall 触发 calldata 截断与 sender spoofing）

**协议/组件**：受影响代币 TIME（源于第三方合约模板生态中的 ERC-2771 + Multicall 组合模式）。`cite: turn8view0, turn26view0, turn6view0`  
**事件日期**：2023-12-07。`cite: turn6view0, turn8view0, turn26view0`  
**损失金额**：约 84.59 ETH 级别（OpenZeppelin 观测到的“野外攻击”金额），CertiK 报告口径为攻击造成约 89.5 ETH 损失、攻击者获利约 84.6 ETH（约 188K 美元）。`cite: turn8view0, turn31search5, turn6view0`  

**被利用函数/模块**：  
- Forwarder 的 `execute()`（验证 `req.from` 签名后拼接 calldata 并 `call` 目标）；`cite: turn6view0, turn26view0`  
- 代币合约的 `multicall(bytes[])`（对每个子调用 `delegatecall`）；`cite: turn6view0, turn26view0`  
- ERC-2771 的 `_msgSender()`（从 calldata 末尾 20 字节提取“原始 sender”）。`cite: turn8view0, turn26view0`  

**根因（同族：认证 from 与实际 _msgSender 解析不一致）**：  
Forwarder 设计意图是把已验签的 `req.from` 作为 calldata 后缀传给目标合约，以便目标合约基于 ERC-2771 从 calldata 末尾恢复真实 sender。实际实现中，Forwarder 直接 `abi.encodePacked(req.data, req.from)`，但当 `req.data` 本身调用的是 `multicall(bytes[])` 时，ABI 解码会按 bytes[] 的 offset/length 截取子元素，导致拼接在末尾的 `req.from` 可能被 multicall 参数解析**截断**，不再处于预期位置。随后 multicall 用 `delegatecall` 执行 `burn()` 等子调用，使 `msg.sender` 保持为 forwarder（满足 `isTrustedForwarder`），而 `_msgSender()` 从“实际 calldata 末尾”读出的 20 字节，变成攻击者可控的地址（例如 AMM 池地址），从而以错误主体执行 burn/transfer 等敏感逻辑。`cite: turn6view0, turn8view0, turn26view0`  

**攻击路径（TIME 事件可复核步骤）**：  
1) 先获得 TIME 代币并准备攻击目标池。`cite: turn6view0, turn26view0`  
2) 通过 Forwarder.execute 调用代币合约 multicall，multicall delegatecall 执行 burn，把池子地址的大量 TIME 直接烧毁，导致价格畸变。`cite: turn6view0, turn26view0`  
3) 利用价格畸变反向换出 WETH/ETH 获利。`cite: turn6view0, turn26view0`  

**攻击交易或地址**：  
- 攻击交易：0xecdd111a60debfadc6533de30fb7f55dc5ceed01dfadd30e4a7ebdb416d2f6b6。`cite: turn6view0, turn26view0, turn8view0`  
- 攻击者地址：0xfde0d1575ed8e06fbf36256bcdfa1f359281455a；攻击合约：0x6980a47bee930a4584b09ee79ebe46484fbdbdd0。`cite: turn26view0, turn6view2`  

**主要参考来源**：OpenZeppelin 公共披露（含真实野外攻击交易与金额）、CertiK 事件分析、SlowMist 深度分析（含攻击步骤与地址）。`cite: turn8view0, turn6view0, turn26view0`  

---

### 案例四：Swopple (Swop) / Polygon 上的 ERC2771+Multicall 野外攻击（同族：sender spoofing，金额较小但结构典型）

**协议/组件**：Polygon 上的 Swopple（Swop）相关合约/池，攻击模式与 OpenZeppelin 披露的 ERC-2771 + Multicall address spoofing 同源。`cite: turn8view0, turn9view0`  
**事件日期**：2023-12-07 08:48:55 UTC（PolygonScan 记录）。`cite: turn9view0`  
**损失金额**：约 17.3K USDC.e（PolygonScan 显示 17,377.381133 USDC.e 从 Uniswap v3 池流向攻击方；OpenZeppelin 披露金额口径约 17,394 USDC）。`cite: turn9view0, turn8view0`  

**被利用函数/模块**：与案例三同一漏洞族——ERC-2771 上下文从 calldata 尾部取 sender，叠加 multicall delegatecall 造成子调用 sender 被伪造。该笔交易被 PolygonScan 直接标注为 “ERC2771 Exploiter”。`cite: turn9view0, turn8view0`  

**根因（同族）**：同样是“可信 forwarder / context suffix”假设被 multicall 的 ABI 截取与 delegatecall 语义破坏，导致 `_msgSender()` 或类似逻辑读取到攻击者可控地址，从而对池子/代币执行非预期的 mint/burn/transfer 路径并获利。`cite: turn8view0, turn9view0`  

**攻击路径（从链上转账侧可复核）**：  
交易中出现从 Null 地址铸造巨量 Swop 代币、并从 Uniswap v3: USDC/Swop 池向攻击相关地址转出约 17,377 USDC.e 的资金流。`cite: turn9view0`  

**攻击交易或地址**：  
- 攻击交易（Polygon）：0x1b0e27f10542996ab2046bc5fb47297bcb1915df5ca79d7f81ccacc83e5fe5e4。`cite: turn9view0, turn8view0`  
- 攻击者（PolygonScan 标注）：0x0a4311b6a2E6DBC5b6A1a0C2bD77B3D83F220a1C。`cite: turn9view0`  

**主要参考来源**：OpenZeppelin 公共披露（列出该 Polygon 交易与金额）、PolygonScan 交易详情页（转账与时间戳可核验）。`cite: turn8view0, turn9view0`  

---

### 案例五：ParaSwap Augustus V6（低相似度同族：回调 caller 校验缺失引发“可伪造执行源”，属于 arbitrary call auth bypass 类）

**协议/组件**：`entity: ["organization","ParaSwap","dex aggregator"]` Augustus V6（合约在以太坊侧的标识地址之一为 0x00000000FdAC7708D0D360BDDc1bc7d097F47439）。`cite: turn11view0, turn12search8`  
**事件日期**：漏洞在 2024-03-20 被确认并公开处置；初始异常交易发生在 2024-03-18~03-20 窗口期，且存在后续“旧授权被利用”的跟进攻击。`cite: turn11view0`  
**损失金额**：postmortem 总结为：初始 exploit 约 $24,000；follow-up attacks 约 $1,100,000；白帽救援约 $3,400,000；并有部分返还与 DAO 覆盖。`cite: turn11view0`  

**被利用函数/模块**：`uniswapV3SwapCallback()` 回调逻辑（用于 gas 优化的 UniswapV3 直连 swap 路径）。`cite: turn11view0`  

**根因（与 ABI smuggling 的关系说明）**：  
这起事件不是典型“ABI offset 走私”，而是更偏“回调信任边界失败”的 arbitrary call auth bypass：`uniswapV3SwapCallback()` 预期仅由真实 Uniswap V3 pool 回调，但在某些情况下缺少/未正确实现 caller check，攻击者可创建“伪造 pool”并触发回调，从而让合约在回调内执行对用户已授权资产的支付/转移逻辑。该模式与 ABI-smuggling 家族的共同点在于：都属于 **“执行源/执行上下文可被伪造，导致授权资产被动支付”**，但差异是：这里的核心不是“ABI 动态 offset 造成解析分歧”，而是“caller 身份绑定失败”。`cite: turn11view0`  

**攻击路径（postmortem 描述）**：  
攻击者通过伪造 pool 调用回调，利用用户对 Augustus V6 的授权在回调中被动扣款，从而盗走用户 token（影响跨多条集成 Uniswap V3 的网络）。`cite: turn11view0`  

**攻击交易或地址（可核验线索）**：  
- 合约地址（Ethereum）：0x00000000FdAC7708D0D360BDDc1bc7d097F47439。`cite: turn12search8`  
- 白帽资金集中地址（Safe Wallet）：0x66E90d840D7C4F3473E25dD8ca361747058c6Db0（用于归集与退款追踪）。`cite: turn11view0`  

**主要参考来源**：Velora（ParaSwap）官方 postmortem。`cite: turn11view0`  

## 根因对照与“ABI smuggling 相似度”归因总结

从“授权检查/上下文绑定使用的字段”是否与“实际执行路径使用的字段”一致出发，可把上述事件映射到你的根因列表：

- **案例一（V4 Router by z0r0z）**：典型“动态 bytes offset + 固定偏移 `calldataload`”导致 Auth/Exec 对同一字段（payer）的解释不一致；属于你列出的“手动解析 calldata 用固定偏移”“ABI 动态参数 offset 导致权限绕过”“generic router 相关解析错误”交集；并满足严格 ABI smuggling 的判定。`cite: turn13view0, turn22view1, turn21view0`  
- **案例二（1inch Fusion V1）**：属于“手动重构 calldata/内存 + 可控长度字段导致指针 underflow”，使 resolver 上下文被伪造，最终形成“arbitrary call auth bypass”；与“selector confusion/offset confusion”同族，但不依赖 ABI 动态 offset 的合法性，而是依赖 assembly 指针算术错误。`cite: turn24view0, turn25view0`  
- **案例三/四（TIME 与 Swopple）**：属于“meta-transaction forwarder + multicall/delegatecall + context suffix 解析错误”，本质是“认证来源（forwarder 验签的 from）与执行来源（token 合约 `_msgSender()` 解析值）不一致”；对应你列出的“meta-transaction forwarder、multicall、proxy/fallback/generic executor 相关的 calldata 解析错误”。`cite: turn8view0, turn6view0, turn26view0, turn9view0`  
- **案例五（ParaSwap）**：更偏“回调 caller 校验缺失导致任意外部触发回调支付”，与 ABI smuggling 的相似性主要体现在“执行上下文可伪造 → 授权资产被动支付”，但其根因核心不是 ABI offset/selector 分歧，因此相似度最低。`cite: turn11view0`  

## 面向防护与审计的可操作检查点

围绕“避免两套解析语义”这一共同主题，以下检查点能直接覆盖本报告中的主要模式（按优先级）：

第一，对任何 `bytes calldata` / `bytes memory` 参与权限判定的路径，强制保证“Auth 与 Exec 使用同一套解析结果”。在案例一中，Auth 用硬编码 `calldataload(164)`，Exec 用 ABI offset 解码，形成可被构造的分歧；因此审计时对 **“固定绝对偏移读取动态参数内容”** 一律视为高危。`cite: turn22view1, turn13view0`  

第二，凡是存在“复制 calldata 到内存并 patch/拼接后再外部调用”的逻辑（类似案例二），都要把“长度字段/offset 字段”当作不可信输入：需要显式边界检查、防止算术 underflow/overflow，并确保写入位置不会回退覆盖先前字段（尤其是 suffix/context 这类安全关键字段）。`cite: turn24view0, turn25view0`  

第三，凡是组合使用 ERC-2771（或任何 calldata 尾部携带 sender 的变体）与 multicall/delegatecall/self-call 的合约，都必须保证：  
- 多重调用层级下 `_msgData()`/`_msgSender()` 的 suffix 长度识别与裁剪是**一致且不可被子调用绕过**；  
- multicall 子调用不得让攻击者控制“末尾 20 字节”成为新的 sender。  
OpenZeppelin 披露的“野外攻击”证明该模式不止一次被用于真金白银攻击。`cite: turn8view0, turn26view0, turn6view0`  

第四，router/aggregator/callback 设计里，所有“会触发支付/转账”的回调函数，都要把 caller 身份与上下文做强绑定（例如严格校验 pool 地址、工厂地址、initcode hash、以及回调参数一致性）。案例五显示，哪怕没有 ABI offset 分歧，只要 caller 校验在某分支缺失，就能变成“伪造执行源”从而盗走授权资产。`cite: turn11view0`  

## 附录：原始链接汇总（每案关键证据）

以下链接均为原始来源（事故复盘/官方披露/区块浏览器/源码），便于你逐条复核。

### 案例一：V4 Router by z0r0z（严格 ABI smuggling）
```text
BlockSec 事件周报（含根因与攻击交易链接）:
https://blocksec.com/blog/weekly-web3-security-incident-roundup-mar-2-mar-8-2026

Etherscan 攻击交易:
https://etherscan.io/tx/0xfe34c4beee447de536bbd3d613aa0e3aa7eeb63832e9453e4ef3999924ab466a

Etherscan 脆弱 Router 合约源码（含 calldataload(164)）:
https://etherscan.io/address/0x00000000000044a361ae3cac094c9d1b14eece97

Etherscan 受害者地址（被冒用 payer）:
https://etherscan.io/address/0x65a8f07bd9a8598e1b5b6c0a88f4779dbc077675

源码仓库（Router 项目）:
https://github.com/z0r0z/v4-router
```

### 案例二：1inch Fusion V1 Resolver（calldata corruption / suffix 注入）
```text
Decurity Postmortem（含根因、受害者/攻击者地址、攻击交易列表）:
https://blog.decurity.io/yul-calldata-corruption-1inch-postmortem-a7ea7a53bfd9

BlockSec 2025 年度事件总结中对该事件的概述:
https://blocksec.com/blog/top-10-awesome-security-incidents-in-2025
```

### 案例三：TIME Token（ERC2771 + Multicall sender spoofing）
```text
OpenZeppelin 公共披露（含野外攻击交易与金额）:
https://www.openzeppelin.com/news/arbitrary-address-spoofing-vulnerability-erc2771context-multicall-public-disclosure

CertiK 事故分析（TIME Token Exploit）:
https://www.certik.com/blog/time

SlowMist 深度分析（含攻击步骤、交易与地址）:
https://slowmist.medium.com/an-in-depth-analysis-of-arbitrary-address-spoofing-attacks-5e6df80d5dae

Etherscan 攻击交易:
https://etherscan.io/tx/0xecdd111a60debfadc6533de30fb7f55dc5ceed01dfadd30e4a7ebdb416d2f6b6
```

### 案例四：Swopple / Polygon（ERC2771+Multicall 野外攻击之一）
```text
OpenZeppelin 公共披露（列出该 Polygon 攻击交易）:
https://www.openzeppelin.com/news/arbitrary-address-spoofing-vulnerability-erc2771context-multicall-public-disclosure

PolygonScan 攻击交易:
https://polygonscan.com/tx/0x1b0e27f10542996ab2046bc5fb47297bcb1915df5ca79d7f81ccacc83e5fe5e4
```

### 案例五：ParaSwap Augustus V6（回调 caller 校验缺失 / fake pool）
```text
Velora（ParaSwap）官方 Postmortem:
https://veloradex.medium.com/post-mortem-augustus-v6-vulnerability-of-march-20th-2024-5df663a4bf01

Etherscan：Augustus V6 合约标识地址之一:
https://etherscan.io/address/0x00000000fdac7708d0d360bddc1bc7d097f47439
```
