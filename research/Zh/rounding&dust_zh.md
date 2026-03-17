# 截至 2026-03-16 的 DeFi／NFTFi rounding & dust 族真实安全事件深度检索报告

## 研究范围与根因家族定义

本报告聚焦一类在链上经常被低估、但在“可组合 + 原子化（atomic）多步调用”环境下会被系统化放大的漏洞家族：**整数除法向下取整、精度截断、舍入方向错误、余数（remainder）归属/清理策略不当**，进而在“份额/记账/兑换率/费用”链路上形成可稳定套利或资产流失的偏差。典型表现包括：

当协议在**缩放（scale）、价格/兑换率换算、份额铸造/销毁、债务份额（debt share）记账、池子不变量（invariant）求解、批量结算（batch）**等路径中，对应当“偏向协议安全”的舍入策略（例如输入侧向上取整、输出侧向下取整、余数归协议）处理不一致，攻击者就能通过**重复小额操作/拆单/循环 mint-burn-redeem / buy-sell / borrow-repay**把本该被丢弃或被协议吸收的误差变成持续收益。

本次输出严格遵循你的区分要求：  
- **严格意义上的 dust / rounding / truncation exploit**：直接利用“舍入/截断/余数处理”产生系统性收益，往往伴随 micro-ops 累积、totalSupply/totalShares 极小、或批量操作放大。  
- **名称不同但根因同类**：例如 tick 计算/流动性双计、solver 数值不稳导致截断歪向、或记账变量/分支条件与舍入方向组合引发的套利与资产流失。

## 方法与筛选标准

- 仅纳入**公开披露且真实发生**（有链上交易/地址或权威复盘确认）的事件；不纳入 CTF、demo、纯理论 PoC、单纯“漏洞赏金但未被利用”。  
- 优先采用你指定的高质量来源：**官方 post-mortem / incident disclosure**、以及 **OpenZeppelin / Trail of Bits / BlockSec / SlowMist / CertiK** 等安全团队的技术分析；并尽可能给出**交易哈希、攻击合约/地址、核心合约地址**等可核验链上证据。

## 高相似度案例库（严格意义 dust/rounding/truncation exploit）

以下按与“dust/rounding/accounting mismatch 根因”的**相似度从高到低**排序（越靠前越贴近“反复微操作累积 dust / 舍入误差、以及 share/totalSupply/内部记账不一致”这条主线）。

### V2 Composable Stable Pools：batchSwap 放大舍入偏差导致超 1.2 亿美元级别损失

**归类**：严格意义的 rounding / precision loss / dust amplification（micro-swaps + batch + 内部记账放大利润）。  

**事件日期**：2025-11-03（多份分析给出约 07:40 UTC 左右开始）。  

**损失金额**：各口径在 **$120M+ 到 $128.64M** 区间；差异源于统计范围（跨链/分叉/后续找回与保护金额）。  

**被利用函数 / 模块（关键链路）**：  
- `Vault.batchSwap`（原子化批量 swap，内部先记账后净额结算）；  
- `ComposableStablePool.onSwap` → `_swapGivenOut`（EXACT_OUT 路径）→ `_upscale` / `_upscaleArray`（将余额按 scalingFactor 缩放）；  
- `_scalingFactors`（引入 exchange rate 后产生**非单位（non-unitary）** scaling factor，使截断更“有利可图”）；  
- 资金以 `InternalBalance`（Vault 内部余额）形式先累积，再通过 `manageUserBalance` 等路径提走（属于“记账变量/内部余额”层面的可利用面）。  

**根因（从“dust/rounding 家族”映射）**：  
- 缩放函数 `_upscale`（或 `_upscaleArray`）在关键路径中采用**单向 round-down（`mulDown`）**，不随 swap 方向区分“输入应上取整、输出应下取整”的安全原则；当金额很小、且 scalingFactor 非单位时，**精度截断会显著低估应付金额/扭曲不变量与 BPT 价格**，微小偏差在 batchSwap 的多轮迭代中被系统性放大。  

**攻击路径（高层复原）**：  
攻击者利用 batchSwap 的“先做多步交换、最后净额结算”的特性，把一次 swap 的“几 wei 级”截断偏差变成可复用的套利引擎：  
1) 先把目标池推到**低流动性/余额接近舍入边界**的状态，使 rounding 误差相对占比更大；  
2) 在单笔 batchSwap 中组织数十次（来源中出现 65+ 次）微操作，让 `_upscale` 的 round-down 偏差在不变量/价格计算中累积；  
3) 利用被压低的 BPT（池份额）隐含价格，**以更低“计价成本”拿到/赎回更高价值资产**；收益先体现在 Vault 的 `InternalBalanceChanged` 等内部记账事件，再在后续交易中提走。  

**攻击交易或攻击者地址（链上证据）**：Check Point 复盘给出较完整的三元角色结构（部署者/攻击合约/接收者）与关键两笔交易（部署期完成盗取、随后提现）。  

**主要参考来源**：  
- Check Point Research 复盘（含 tx、地址、代码路径、internal balance 视角）；  
- OpenZeppelin 技术分析（解释 `_upscale`、`_scalingFactors`、batchSwap 调用链与“amount 很小 + 非单位 scalingFactor”导致截断放大）；  
- Trail of Bits 总结与审计视角（强调“rounding direction issue”的长期性与测试/不变量策略）；  
- BlockSec 深度分析（强调 rounding inconsistency 与攻击流程要点）。  

**如果 rounding 方向/余数归属/dust 策略设计正确，攻击是否还能成立？**  
大概率**不能成立或无法规模化**：若 `_upscale` 按 swap 方向分别 up/down、并对小额/低流动性状态加入约束（最小交换量、舍入偏差上限、不变量变化验证、禁止在同一 batch 中累计偏差），这类“微误差稳定套利”会被直接掐断。  

**原始链接（便于核验）**：
```text
Check Point 复盘：
https://research.checkpoint.com/2025/how-an-attacker-drained-128m-from-balancer-through-rounding-error-exploitation/

关键交易（文中给出）：
TX1 (deploy+theft): 0x6ed07db1a9fe5c0794d44cd36081d6a6df103fab868cdd75d581e3bd23bc9742
TX2 (withdraw):     0xd155207261712c35fa3d472ed1e51bfcd816e616dd4f517fa5959836f5b48569

攻击相关地址（文中给出）：
Deployer:        0x506D1f9EFe24f0d47853aDca907EB8d89AE03207
Exploit Contract:0x54B53503c0e2173Df29f8da735fBd45Ee8aBa30d
Recipient:       0xAa760D53541d8390074c61DEFeaba314675b8e3f

OpenZeppelin 技术分析：
https://www.openzeppelin.com/news/understanding-the-balancer-v2-exploit

Trail of Bits 分析与建议：
https://blog.trailofbits.com/2025/11/07/balancer-hack-analysis-and-guidance-for-the-defi-ecosystem/
```

### yETH：solver 截断归零 + dust 触发初始化分支 + underflow 的复合型无限铸造

**归类**：严格意义的 truncation/rounding + dust + 记账变量不一致（多阶段复合）。  

**事件日期**：2025-11-30（区块 23,914,086）。  

**损失金额**：官方披露中列出池内多种 LST 资产被抽干，并提到后续协助回收 857.49 pxETH；外部报道常概括为约 $9M 级别（不同统计口径）。  

**被利用函数 / 模块（关键链路）**：  
- `add_liquidity` / `remove_liquidity`（LP 进出与供应/不变量更新）；  
- `_calc_supply`（迭代式 fixed-point solver / Newton 迭代近似）；  
- `update_rates` / `_update_supply`（内部 `D` 与 ERC-20 `totalSupply` 的 reconcilliation，且涉及 `st-yETH` 的 POL 吸收损失/收益机制）。  

**根因（从“dust/rounding 家族”映射）**：  
官方披露把 exploit 拆成三阶段，其中与本题最贴合的点有两类：  
- 在 Phase 1 中，solver 的迭代在极端不平衡输入下发生数值不稳，最终出现**`sp / s` 在整数运算下被 round-down 到 0**，导致乘积累积器 `r` 塌缩为 0，从而使 `Π = 0`、不变量退化并触发超额铸造；  
- 在 Phase 3 中，池被清空后 `prev_supply == 0`，攻击者以“dust 配置”重新进入初始化分支，并触发类似 `unsafe_sub(A*Σ, D*Π)` 的算术 underflow，导致巨量 LP 铸造（属于“边界分支 + dust + 算术安全”组合问题）。  

**攻击路径（官方三阶段摘要）**：  
1) **Phase 1**：构造极端不平衡的 `add_liquidity` 序列，把 solver 推入非预期域；在迭代中由于整数截断，关键比值 round-down 到 0，`r → 0` 进而 `Π → 0`，协议在缺乏域校验的情况下接受错误的 `D` 并超额铸造 LP；  
2) **Phase 2**：调用 `remove_liquidity(0)` 等路径“修复” `Π`，但不纠正被抬高的 `D`；随后利用 permissionless 的 `update_rates` 触发 `_update_supply`，把“差额”当作系统 slashing，通过燃烧 `st-yETH`（POL）来对齐供应，从而**把超铸成本转嫁给 POL 而不是攻击者**；重复循环提走 LST；  
3) **Phase 3**：在池被抽空后，利用 dust 重新进入初始化分支并触发 underflow，铸造约 2.3544×10^56 的 LP，再进一步抽干下游 yETH/ETH 池。  

**攻击交易或攻击者地址（链上证据）**：官方 disclosure 给出 exploit tx 与合约地址引用（含 BlockSec Explorer 链接）。  

**主要参考来源**：Yearn 安全披露（官方、含阶段划分、合约/tx 引用与时间线）。  

**如果 rounding 方向/余数归属/dust 策略设计正确，攻击是否还能成立？**  
大概率**不能成立**：若对 solver 域（domain validity）与收敛条件做链上强校验、避免关键比值在整数截断下归零、并在 `prev_supply==0` 等初始化分支对状态/参数设置强约束（以及对 underflow 使用安全算术），攻击链会在 Phase 1 或 Phase 3 被切断。  

**原始链接（便于核验）**：
```text
Yearn 官方 Incident Disclosure（含合约与 tx 引用）：
https://github.com/yearn/yearn-security/blob/master/disclosures/2025-12-01.md

yETH Weighted Stableswap Pool 合约（disclosure 引用）：
https://etherscan.io/address/0xCcd04073f4BdC4510927ea9Ba350875C3c65BF81

Exploit Transaction（disclosure 引用 BlockSec Explorer）：
https://app.blocksec.com/explorer/tx/eth/0x53fe7ef190c34d810c50fb66f0fc65a1ceedc10309cf4b4013d64042a0331156
```

### ：用“2 wei totalSupply + 捐赠操纵 exchangeRate + truncate 向下取整”抽干市场

**归类**：严格意义的 dust（极小 totalSupply）/ rounding down / exchange-rate manipulation（empty-market donation）组合。  

**事件日期**：2023-04-16。  

**损失金额**：约 **$7.4M**。  

**被利用函数 / 模块（关键链路）**：  
- `mint()` / `redeem()`（Compound v2 fork 的 hToken/soToken 类逻辑）；  
- `exchangeRate` 计算（`getCash()`、`totalSupply()` 等变量组合）；  
- `truncate()` / 向下取整导致的精度损失（本质是“应 burn 的份额稍小于整数门槛 → floor 后少 burn 1 单位”）。  

**根因（从“dust/rounding 家族”映射）**：  
BlockSec 指出攻击核心在于：在 `redeem()` 过程中，理论应扣减的 hToken 份额接近 2（如 1.99999992），但因为 `truncate()`/整数化使用**rounding down**，最终只扣 1，从而出现可被系统化利用的“少扣份额”偏差；而当市场处于**empty/low-liquidity（特别是 totalSupply 被人为压到接近 0）**时，攻击者还能通过“直接转账捐赠”操纵 `getCash()`，把 exchangeRate 放大到极端，令这 1 个份额的“少扣”对应巨大资产价值。  

**攻击路径（按 BlockSec 复盘摘要化）**：  
攻击者通过闪电贷准备资金后，刻意把 hWBTC 市场的 `totalSupply` 做到 **2 wei**：先 mint 一些再 redeem 到只剩极小余量；随后把大量 WBTC **直接转账到市场合约**（不走 mint，因而不增发 hWBTC），使 `getCash` 巨大而 `totalSupply` 极小，exchangeRate 数百倍放大；最后利用 redeem 扣减份额时的 truncation，使“应扣 2 却只扣 1”的差额变成可兑现的大额资产，并进一步扩展到其它市场抽干。  

**攻击交易或攻击者地址（链上证据）**：BlockSec 给出 Optimism 上攻击交易哈希。  

**主要参考来源**：BlockSec 事件复盘（包含根因解释与分步流程）。  

**如果 rounding 方向/余数归属/dust 策略设计正确，攻击是否还能成立？**  
大概率**不能成立**：若 redeem 计算在安全侧进行（例如对“应 burn 的份额”向上取整、或对 exchangeRate 极端条件做约束）、并通过“种子流动性/最小 totalSupply 不为 0、禁止或正确处理 donation 影响”消除 empty-market 条件，攻击面会消失。  

**原始链接（便于核验）**：
```text
BlockSec 复盘：
https://blocksec.com/blog/6-hundred-finance-incident-catalyzing-the-wave-of-precision-related-exploits-in-vulnerable-forked-protocols

攻击交易（BlockSec 文中给出哈希，链为 Optimism）：
0x6e9ebcdebbabda04fa9f2e3bc21ea8b2e4fb4bf4f4670cb8483e2f0b2604f451

参考打开：
https://optimistic.etherscan.io/tx/0x6e9ebcdebbabda04fa9f2e3bc21ea8b2e4fb4bf4f4670cb8483e2f0b2604f451
```

### ：Compound fork “donation + 极小 supply + truncation”复现，约 2000 万美元级损失

**归类**：严格意义的 precision loss / truncation + empty-market donation manipulation（与 Hundred Finance 同一“祖谱”）。  

**事件日期**：2024-05-14（多方以此日期描述事件爆发；亦有报道在 5/15 继续追踪资金流）。  

**损失金额**：约 **$20M**。  

**被利用函数 / 模块（关键链路）**：  
- Compound v2 fork 的 soToken 市场创建/初始化与 `exchangeRate` 逻辑（empty market + 极小 totalSupply）；  
- `borrow()`、`getAccountSnapshot()`、以及 collateral 计算中的截断（truncation）偏差；  
- “donation”（直接转账到 soToken 合约导致 `totalCash` 增大而不增发 soToken）。  

**根因（从“dust/rounding 家族”映射）**：  
CertiK 明确把 Sonne 事件归为“已知 precision loss 漏洞（Hundred Finance 同类）在空市场/新市场情形下被再次利用”。  
SharkTeam 进一步给出关键机制：攻击者通过闪电贷把大量 VELO **直接转入 soVELO 合约**，由于不走 mint，soVELO `totalSupply` 不变而 `totalCash` 激增，exchangeRate 被推高；随后在借贷与赎回链路中，因**截断（truncation）**导致的抵押需求低估与份额兑换偏差，实现以“极小 soVELO”撬动大额借款/赎回。  

**攻击路径（按 SharkTeam 复盘概要）**：  
1) 闪电贷获取约 35,569,150 VELO；把 VELO 直接转入 soVELO（donation），totalCash ↑、totalSupply 不变；  
2) 部署/使用攻击合约，将少量 soVELO 作为抵押，声明 soWETH/soVELO 为抵押品与借贷市场，借出 WETH；  
3) 利用计算中的 truncation，使“应需要的抵押/份额”被低估，并可在赎回时取出此前 donation 的大额 VELO；重复多次扩大收益；  
4) 归还闪电贷，保留获利资产。  

**攻击交易或攻击者地址（链上证据）**：SharkTeam 给出攻击交易哈希。  

**主要参考来源**：  
- CertiK 事件分析（将其归入已知 precision loss/Compound fork 家族）；  
- SharkTeam 交易级复盘（给出 donation→借贷→重复的可操作链路与 tx hash）；  
- Halborn 复盘（强调“分步、permissionless 执行”给攻击者可乘之机，以及同类 donation 攻击逻辑）。  

**如果 rounding 方向/余数归属/dust 策略设计正确，攻击是否还能成立？**  
大概率**不能成立**：若新市场上线前强制 seed 足够的初始 supply（避免 dust totalSupply）、将 donation 计入并同步增发份额或直接禁止、并在抵押/兑换/borrow 链路对截断偏差做安全侧舍入（向上取整抵押需求、向下取整可借额度），攻击链会被破坏。  

**原始链接（便于核验）**：
```text
CertiK 分析：
https://www.certik.com/blog/sonne-finance-incident-analysis

SharkTeam 分析（含攻击 tx hash）：
https://medium.com/@sharkteam/sharkteam-analysis-of-sonne-finance-attack-incident-b57a4529c475
攻击交易（文中出现）：
0x9312ae377d7ebdf3c7c3a86f80514878deb5df51aad38b6191d55db53e42b7f0

Halborn 复盘：
https://www.halborn.com/blog/post/explained-the-sonne-finance-hack-may-2024
```

### ：指数被放大 + divUp 舍入导致“1 wei 抵押铸 1 wei 份额”，可迭代累积并撬动大额借款

**归类**：严格意义的 precision/rounding（share minting）+ “小额重复操作累积”放大；同时伴随业务逻辑缺陷放大指数。  

**事件日期**：2023-11-10（官方复盘给出 18:59:23 UTC）。  

**损失金额**：官方描述为约 **$6.7M unbacked R** 被铸造并卖出导致脱锚；部分分析以换得 ETH 规模描述（并出现攻击者误烧 ETH 的逆转细节）。  

**被利用函数 / 模块（关键链路）**：  
- `InterestRatePositionManager`（IRPM，官方点名被利用合约）；  
- share minting 中的 `divUp` 舍入（round-up）行为，使极小输入也能铸出 1 单位份额；  
- 指数（index / storedIndex）因 donation/清算路径被异常抬高，扩大“1 份额”的价值表达。  

**根因（从“dust/rounding 家族”映射）**：  
官方 post-mortem 把“主要根因”描述为：**铸造 share token 时的精度计算问题**使攻击者获得额外份额；当指数被放大后，攻击者可以用极小份额换取大额 cbETH，再用放大的抵押能力借出大量 R。  

**攻击路径（官方逐步列举）**：  
1) 闪电贷获得 6,000 cbETH；  
2) 向 IRPM 转入总计 6,001 cbETH，并对预先创建的仓位触发清算；  
3) 通过 donation 使抵押指数被放大上千倍；  
4) 利用 `divUp`：用 **1 wei cbETH 铸 1 wei share**，重复 60 次，把“dust 级输入”累积成可观抵押表达；  
5) 用极少 rcbETH-c 赎回大额 cbETH，并据此借出约 6.7m R；再在多个池子兑换为 ETH 等资产；复盘还记录了攻击者误操作烧毁 1,570 ETH 的细节。  

**攻击交易或攻击者地址（链上证据）**：官方 post-mortem 给出 exploit tx 与攻击者地址。  

**主要参考来源**：Raft 官方 post-mortem（含 tx/地址/步骤/根因），以及 DN Institute 的摘要索引。  

**如果 rounding 方向/余数归属/dust 策略设计正确，攻击是否还能成立？**  
大概率**不能成立**：若 share mint 对极小输入采用安全侧舍入（通常是 floor/最低铸造门槛/最小抵押单位），并确保指数不可被“捐赠/清算路径”放大到非经济合理区间，那么“循环 1 wei 铸份额”无法形成可兑现价值。  

**原始链接（便于核验）**：
```text
Raft 官方 post-mortem（含 exploit tx / attacker / 合约）：
https://paragraph.com/%40raft/raft-security-incident-post-mortem-analysis-and-recovery-plan

Exploit Transaction（官方文中给出）：
https://etherscan.io/tx/0xfeedbf51b4e2338e38171f6e19501327294ab1907ab44cfd2d7e7336c975ace7

Attacker（官方文中给出）：
https://etherscan.io/address/0xc1f2b71a502b551a65eee9c96318afdd5fd439fa

Exploited Contract（官方文中给出）：
https://etherscan.io/address/0x9ab6b21cdf116f611110b048987e58894786c244
```

### ：rebase 记账在“elastic=0、base≠0”边界下失配，循环 borrow/repay 利用 rounding 偏差低估债务

**归类**：严格意义的 fee/debt accounting rounding mismatch（看似不是 dust，但本质属于“记账变量在舍入/边界条件下失配”）。  

**事件日期**：2024-01-30。  

**损失金额**：约 **$6.4M–$6.5M**（不同媒体/统计口径略有差异）。  

**被利用函数 / 模块（关键链路）**：  
- `Cauldron v4`（多篇报道指向 v4 cauldrons）；  
- “rebase library” 同步 `elastic` 与 `base` 的记账逻辑；漏洞表现为 **`elastic == 0` 但 `base` 非 0 时未正确处理差异**，为循环操作留下可套利的 rounding 缝隙。  
- 报道中提到攻击者反复调用 `userBorrowPart()` 并伴随 `repay()` 循环（体现你要求的“循环小额操作放大”特征）。  

**根因（从“dust/rounding 家族”映射）**：  
Blockworks 引述的分析指出：本应同步 `elastic`/`base` 的 rebase 记账在边界条件下失配，攻击者通过反复利用 rounding 误差把系统对债务的估计“做小”，实现“实际借得多、偿还计价少”的结构性收益。  

**攻击路径（公开信息可复原的稳定部分）**：  
1) 在特定 Cauldron（v4）中，通过 borrow/repay 的多次循环触发 rebase 同步缺陷；  
2) 由于 `elastic` 与 `base` 在某些状态下不一致（且处理逻辑未吸收/隔离余数），系统逐步低估债务；  
3) 最终实现抽取 MIM 的净流出，并在链上换成 ETH 等资产。  

**攻击交易或攻击者地址（链上证据）**：Blockworks 指向一笔“suspect transaction”，其哈希在报道中可见（但链上细节需在浏览器核验）。  

**主要参考来源**：  
- Blockworks 报道（明确“rounding bug / rebase library / elastic=0 base≠0”叙述，并给出可追踪交易）；  
- The Block 报道（给出 $6.4M 量级损失的新闻口径）；  
- 其它聚合报道引用 CertiK 的“rounding issue + borrow/repay loop”判断（用于补强攻击链路描述）。  

**如果 rounding 方向/余数归属/dust 策略设计正确，攻击是否还能成立？**  
大概率**不能成立**：若在 rebase/债务份额记账中强制维护不变量（`base==0 ⇔ elastic==0` 或给出严格转换规则）、把余数系统性归入协议（或累积到专用 remainder 账户）、并对“重复 borrow/repay”做最小单位/频次/状态机约束，循环放大将失效。  

**原始链接（便于核验）**：
```text
Blockworks 报道（含“suspect transaction”引用）：
https://blockworks.com/news/rounding-exploit-magic-internet-money

The Block 报道：
https://www.theblock.co/post/275072/abracadabra-finance-drained-of-estimated-6-4-million-in-apparent-security-attack
```

## 同根因家族但“名称不同”的事件

### Elastic：错误舍入方向导致 tick 计算偏移与流动性双计，进而被闪电贷路径抽干

**归类**：同族事件（rounding direction flaw），但攻击表面更像“tick/流动性机制”而非传统 dust；本质仍是**舍入方向错误 → 状态判断偏移 → 记账/流动性被双计**。  

**事件日期**：2023-11-23。  

**损失金额**：  
- BlockSec 摘要口径：总损失 **over $48M**；  
- SlowMist（MistTrack）口径：跨链合计 **over $54.7M**。  
（差异主要来自统计范围、跨链分笔与后续追踪口径。）  

**被利用函数 / 模块（关键链路）**：  
- `computeSwapStep`（swap while-loop 中的关键数学步骤）；  
- `estimateIncrementalLiquidity` / `deltaL` 计算：注释语义期望 `deltaL` round-up 来保证 `nextSqrtP` round-down，但实现误用 `mulDivFloor` 导致 **deltaL round-down**，使 `nextSqrtP` 被错误 round-up；  
- 错误的 `nextSqrtP`/tick 变化判断最终引发“tick 未跨越但价格已越界”的状态，从而造成**流动性双计**并使反向 swap 获利。  

**根因（为何属于同一家族）**：  
这一事件的“可利用性”不是来自传统重入/预言机，而是来自**数学实现与注释语义的舍入方向不一致**。攻击者用极其精细的输入构造，让“只差 1 个最小单位”的舍入差异改变 tick/流动性更新分支，最终实现对流动性的双计与可兑现套利。  

**攻击路径（按 BlockSec 六步概要）**：  
攻击者通过闪电贷借入 WETH，把价格推到“当前无 base liquidity 但有 reinvest liquidity”的区域；再通过加/减流动性把区间与总量调到可控状态；随后用一笔 swap 把池子“骗到”认为 tick 未跨越但实际 sqrtP 已越界，再做反向 swap，此时流动性被双计，从而把一次舍入偏差变成可兑现利润。  

**攻击交易或攻击者地址（链上证据）**：  
- HackMD 汇总给出一笔典型攻击交易、攻击者地址、攻击合约与受影响池合约；  
- SlowMist 给出多条攻击者地址清单（跨链追踪口径）。  

**主要参考来源**：BlockSec 技术复盘与 SlowMist 资金追踪。  

**如果 rounding 方向/余数归属/dust 策略设计正确，攻击是否还能成立？**  
大概率**不能成立**：只要把 `deltaL` 的舍入方向修正为注释语义（ceil），并在“tick 不跨越”的分支强约束 `nextSqrtP` 不得超过下一 tick 的 `sqrtP`、以及在状态更新上避免“双计”可能性，攻击的关键拐点会消失。  

**原始链接（便于核验）**：
```text
BlockSec 复盘（摘要版）：
https://blocksec.com/blog/kyberswap-incident-masterful-exploitation-of-rounding-errors-with-exceedingly-subtle-calculations

SlowMist（MistTrack）复盘（含多条攻击者地址）：
https://slowmist.medium.com/a-deep-dive-into-the-kyberswap-hack-3e13f3305d3a

HackMD 汇总（含典型攻击 tx / attacker / pool 合约）：
https://hackmd.io/@nomorecaffeine/BkHxa9Ta1e
攻击交易（文中出现）：
https://etherscan.io/tx/0x485e08dc2b6a4b3aeadcb89c3d18a37666dc7d9424961a2091d6b3696792f0f3
攻击者地址（文中出现）：
0x50275E0B7261559cE1644014d4b78D4AA63BE836
```

## 场景的匹配度与检索结果

在本次截至 2026-03-16 的公开资料检索中，我**未能找到**满足你“必须是真实发生、且有高质量复盘与链上证据、且核心根因明确属于 rounding/dust/余数处理不当”的**分片 NFT（fractional NFT）市场结算 / NFTFi 赎回分片 / 订单撮合 partial fill 舍入**类攻击事件，能够达到与上述 DeFi 案例同等的证据强度与细节完备度。

这并不意味着 NFTFi 不存在此类风险，而更像是：  
- NFTFi 的公开高额事件更常见于授权/签名、抵押品绕过、预言机/地板价操纵、清算逻辑、重入/回调等；  
- “dust/rounding 族”在 NFTFi 里往往表现为**小额漏损或边界条件 bug**，如果未造成大额损失，团队可能以补丁/赔付而非公开深度复盘的方式处理，导致公开证据链不如主流 DeFi AMM/借贷事件完整。  

若你希望把搜索进一步“逼近分片 NFT 主题”，建议把筛选条件从“必须高额损失”放宽到“确实造成资金差错/超额领取（即使金额较小）”，并把目标扩展到：  
- 分片 NFT 赎回合约的 `previewRedeem`/`redeem` 一致性；  
- 拍卖/撮合的 partial fill remainder 分配；  
- 版税/费用分发的余数累计与可提取边界。  
（但这会触及你当前第 1 条要求的边界：很多会变成“漏洞发现/修复”而非“已被利用的真实攻击”。）

## 从真实事件抽象出的防护要点与可操作判定

这些事件共同指向一个工程现实：在 DeFi/NFTFi 中，**舍入不是“实现细节”，而是“经济安全边界”**。  

可直接映射到你列举根因的防护策略如下（每条都对应至少一个已发生事件证据）：

舍入方向必须与资产流向绑定，而不是“统一 roundDown 就安全”。Balancer 事件说明：当缩放因子包含 exchange rate（非单位），统一 roundDown 会在 EXACT_OUT 等路径下系统性低估应付金额，并可被 batch/micro 操作放大。  

必须显式定义“余数归属”与“dust 清理策略”，并在状态迁移边界清零或收敛。Yearn yETH 事件显示：当系统进入 `prev_supply == 0` 等初始化/重置分支时，若允许 dust 触达该分支或存在 underflow 面，攻击者可以把边界状态变成“无限铸造入口”。  

避免 totalSupply/totalShares 极小或为 0 的可达状态，或对其建立可证明的安全不变量。Hundred/Sonne 这类 Compound fork 反复证明：**empty market + donation + 截断**会把 tiny supply 变成操纵 exchangeRate 的杠杆点。  

对“循环小额操作”要有显式的经济边界：最小铸造/赎回单位、最小 swap 输入/输出、最大迭代次数、以及“累计余数账本”。Raft 与 Abracadabra 都体现了：当 1 wei 级操作能改变份额/债务份额，攻击者就能靠循环把误差变成资产。  

对“多步原子化”入口（batchSwap、cook、多动作 router）必须以**组合攻击**视角测试：单步正确≠组合正确。Balancer 与 KyberSwap 都是“单步看起来误差微小，但组合后进入新状态空间”并最终可盈利的范例。  

最后，一个非常实用的判定准则：如果你能构造一条路径让系统在某个环节出现**“应该扣 2 单位却扣 1 单位”**、或**“should be zero but not zero”**、或**“rounding bias 单向累积”**，并且该偏差可以被循环/批量放大，那么它几乎必然属于你列举的这一家族，应当按“可被系统性套利”优先级处理。