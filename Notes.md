---
typora-copy-images-to: ./image/uniswapV3OverView.png
---

# [Hackathon]

### [Spark AI Hackathon]

**time:** 2026:: 01/26 -> 02/01

**Description:** 我打算做一个Solidity Contests的汇总网站，类似于 [DailyWarden](https://www.dailywarden.com/) 。列出当前正在进行的Competitive Audit和即将到来的Audit的同时，我打算让AI综合对应项目的SLOC和对应推特帖子的数量和讨论度，给出一个全面评价，主要涉及的方面有 1.难度 2.类型 3.参与热度  [我觉得这是一个创新点，目前市场上是没有同类型的产品的]。这个项目的目的是为了方便合约安全审计工程师合理安排比赛时间和任务，节约时间，成为所有安全审计人员的必备工具。

# [EThpanda Weekly Meeting]

### [1/19/2025]

#### 快讯

- 以太坊基础设施基本完成，需要开始走向去中性化世界。是否意味着下一波Dapp风潮
- BNB Chain升级，finalize小于1s
- 市场正在进入一个由政策信号主导的新阶段
- 黑客攻击增加
- starknet只有市值没有交易量
- avalanche的c-chain活跃性达到历史新高

#### 以太坊新EIP

下次升级主要feature: BAL/ePBS

- EIP7778
- EIP7708 ETH转账会发送log
- EIP7843 SLOTNUM Opcode 通过block时间戳计算slot。
- EIP8024 加入SWAPn和DUPn Opcode

#### Tempo链稳定币支付

稳定币绕过中间商(银行，汇率差)直接交付，在跨国交易上有优势

- 实时结算
- 低成本
- 可追踪

# [Basic]

### [B-1] Etherscan::Transaction

**Description** 对于 ETH 来说，区分交易的类别是很重要的。

#### [Transaction type classification]

- **🛜Type0 [Legacy Transaction]**

  最早的以太坊交易形式，使用 **单一 Gas Price**，没有 Base Fee / Priority Fee 的概念，手续费 = `Gas Used × Gas Price`，现在仍然**兼容**，但不推荐使用。这也是很多链下服务出 bug 的一个点，协议太老了，不适配新协议。下面的表格着重看 gasPrice 和 gasLimit

  | 字段     | 说明          |
  | -------- | ------------- |
  | gasPrice | 固定 gas 单价 |
  | gasLimit | Gas 上限      |
  | nonce    | 交易序号      |
  | to       | 接收地址      |
  | value    | ETH 数量      |
  | data     | 合约数据      |

- 🛜 **Type1 [Access List Transaction (EIP-2930)]**

  核心概念是 AccessList，这个会直接声明要访问的数据

  ```yaml
  [
    { address: 0xContractA, storageKeys: [slot1, slot2, ...] },
    { address: 0xContractB, storageKeys: [...] },
  ]
  ```

- 🛜 **Type 2：EIP-1559 Transaction (主流)**

  引入 **Base Fee（销毁）**，引入 **Priority Fee（矿工小费）**，自动退还多余 Gas

  ```js
  effectiveGasPrice =
  min(
    maxFeePerGas
    baseFee + maxPriorityFeePerGas
  )
  ```

  | 字段                 | 含义                 |
  | -------------------- | -------------------- |
  | maxFeePerGas         | 你愿意支付的最高 Gas |
  | maxPriorityFeePerGas | 给矿工的小费         |
  | baseFee              | 网络自动决定         |

  相当于原来的`gasPrice`被拆分成了`maxFeePerGas`和`maxPrioityFeePerGas`，实际的 gas Fee

- 🛜 **Type 3：Blob Transaction（EIP-4844 / Proto-Danksharding）**

  2024 年引入，面向 layer2，数据放在 blob 中，隔一段时间主网会删除 Blob

  Rollup（如 Arbitrum、Optimism）提交数据，数据放在 **Blob** 中，而不是 calldata，极低的数据成本，不直接参与 EVM 执行，专为扩容设计。给 Rollup（如 Arbitrum、Optimism）提交数据，数据放在 **Blob** 中，而不是 calldata。极低的数据成本，不直接参与 EVM 执行，专为扩容设计

### [B-2] thoughts about ERC7962

对于”Gas Costs”这一部分我有些想法。一个Dapp如果使用免gas fee的模式必然需要引入”验证层”，否者无法抵抗恶意的Sybil attack导致Dapp的资金被恶意消耗。目前的验证层也许可以是传统的KYC，也可以是还在研发阶段的zk-id，亦或是Sam Altman弄的World-ID。而non-gas-fee模式的引入也是为了更加友好的UX。这样的non-gas-fee + identity Layer的模式也许会是下一轮dapp的风潮。

### [B-3] ISO-4217

ISO 4217 is the international standard defining three-letter alphabetic and three-digit numeric codes for representing currencies, funds, and precious metals, used globally in banking, trade, and finance to avoid ambiguity (e.g., USD for US Dollar, EUR for Euro). It provides clarity by linking countries (first two letters, often from [ISO 3166](https://www.google.com/search?q=ISO+3166&oq=ISO+4217&gs_lcrp=EgZjaHJvbWUqBwgAEAAYgAQyBwgAEAAYgAQyBwgBEAAYgAQyBwgCEAAYgAQyBwgDEAAYgAQyBwgEEAAYgAQyBwgFEAAYgAQyBwgGEAAYgAQyBwgHEAAYgAQyBwgIEAAYgAQyBwgJEAAYgATSAQc3MzJqMGo3qAIAsAIA&sourceid=chrome&ie=UTF-8&mstk=AUtExfB1DwnqVHHfWw032QSB61ys9eh0hnigvB9s_iOFVn2ar_kZXd8aLrIkYLd7Khmko-tn25KR7sLmbTCwm0xoCc671fEBfRv3PA0CNQ4bWEWTlJ_hKRBT8_LHbxnDec4NxOkRoiUqQXonByOe4UkxZJlPxArzbObyrPaQXsZ18XT-2dY&csui=3&ved=2ahUKEwia1qfkvJmSAxU-ho4IHXyLAxcQgK4QegQIARAB)) with their specific currency (third letter), and also details minor currency units and gold (XAU) or silver (XAG).

### [B-4]JIT Attack

**Description:** `JIT`指 `Just In Time`，攻击者在某个关键状态即将生效前或刚生效时，瞬时介入（进入 / 退出 / 修改状态），从而获得不成比例的收益或特权，且几乎不承担对应风险或成本。本质是收益与风险的不对称。下面我举几个例子：

**AMM:** 在DEX的语境下，JIT常常指beforeswap添加大量流动性，afterswap移除流动性，从中赚取Swap Fee的同时不用承担风险。对交易进行排序获得优势。

**FLASH LOAN:** 在借贷的语境下，同上。在一个闪电贷前提供资金，赚取手续费而不承担风险。

总而言之，造成JIT的原因在于合约没有对于流动性做监控。如何防止JIT呢？

**TimeLock:** 对于新加入的流动性，需要Lock一段时间才能参与收益的分配，类似Euler。

**Before Distribution: ** 将受益的分配前置。在Uniswap V4中可以设置 beforeAddliquidity，让攻击者的流动性在添加的时候，旧流动性都能分配给LP持有者。在swap进行的时候，所有的累积收益已经分配完，新流动性无法套利。

**Linear Vesting:** 让收益的分配不只看"钱"，而是用时间来加权，这样可以保证JIT攻击加入的流动性的权重是很低的，无法套利。

# [PANPTIC]

### [SK-PANOPTIC-1] Use BytesMask for more efficient storage

**Description:** `BytesMasking` is a technique to pack multiple values into a single storage slot (usually taking uint256 -> 32 bytes == 256 bits) to save gas, instead of using separate storage slots for each variable (such as struct)

<details>
<summary>💹Example in Panoptic</summary>

```js
PACKING RULES FOR A MARKETSTATE:
From the LSB to the MSB:
(0) borrowIndex          80 bits : Global borrow index in WAD (starts at 1e18). 2**80 = 1.75 years at 800% interest
(1) marketEpoch          32 bits : Last interaction epoch for that market (1 epoch = block.timestamp/4)
(2) rateAtTarget         38 bits : The rateAtTarget value in WAD (2**38 = 800% interest rate)
(3) unrealizedInterest   106 bits : Accumulated unrealized interest that hasn't been distributed (max deposit is 2**104)
Total                    256 bits  : Total bits used by a MarketState.
```

The `MarketState` packs 4 values into 1 storage slot:

|<--0-79-->|<--80-111-->|<--112-149-->|<--150-155-->|

| 80 Bits | 32 Bits | 32 Bits | 106 Bits |

</details>

<details>
<summary>💹How to use this powerful skill?</summary>

Example:
The `MarketState` packs 4 values into 1 storage slot:

|<--0-79-->|<--80-111-->|<--112-149-->|<--150-155-->|

| 80 Bits | 32 Bits | 38 Bits | 106 Bits |

First, define the mask we need. In this case, rateAtTarget is required

```js
TARGET_RATE_MASK = ((1 << 38) - 1) << 112;
// creates: 111...111(38 ones) at position 112-149
// 0x...3FFFFFFFFF000000000000000000000000000
```

Then, we use Yul to load specific storage

- Write Value by mask

```js
MarketState self,
uint40 newRate

assembly{
    //clear bits 112-149
    let cleared := and(self, not(TARGET_RATE_MASK));
    // ...000..000 (38 zeros in position)

    //2. Mask the input to ensure it fits 38 bits
    // uint40 -> we have to ignore the top 2 bits
    // 0011 1111 1111 ....  1111 1111
    let safeRate := and(newRate, 0x3FFFFFFFFF);


    let result := or(cleared,shl(112, safeRate));
}

```

- Read Value by mask

```js
MarketState self

assembly{
    // push->[xxx ...|         RATE          |]
    //               0011 1111 .... 1111 1111
    //               &&&& &&&& &&&& &&&& &&&&
    let result := and(shr(112,self), 0x3FFFFFFF);
}

```

</details>

<br>

**Benefits:**

1. Gas Savings: SSTORE (~20k gas) vs multiple SSTORE
2. Atomic Update: All values update together
3. Cache Efficiency: Reading multiple values costs less

# [Uniswap Introduction]

## Uniswap V2

# Uniswap Introduction

## Uniswap V2

### [UNIV2-1] 为什么需要两个 codebase？

**Discription:** uniswap V2 有两个仓库，`v2-core`和`v2-periphery`。区分二者的重点在于面向对象的不同。v2-core 是核心，里面包含了 pool 的创建，token swap 逻辑，其中的 function 普通用户是用不上的。v2-periphery 专门用用来与用户交互。

### [UNIV2-2] Swap Token in V2

**Discription:** `v2-periphery`提供接口给用户 swapTokens，分别是`UniswapV2RouterV2::swapExactTokensForTokens()`以及`UniswapV2RouterV2::swapTokensForExactTokens()` .

- swapExactTokenForToken() 目的是通过 exactInput -> 计算出 calculated output，然后交易。 举个例子，我有 1WETH，我要用这 1WETH 去兑换 DAI，在这个场景下我不知道我能兑换多少 DAI，但是我会提供指定的 WETH。由于兑换出的 DAI 是未知数，uniswapv2 提供了滑点保护(slip protection)用于抵抗 MEV,说人话就是我(用户)可以指定一个最低兑换 DAI 的数量，如果小于这个数就放弃。
- swapTokenForExactToken() 目的是通过 exactOutput -> 计算出 calculated input，然后交易。

<details>
<summary>💹Swap?TokenFor?Token</summary>

```js
function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external virtual override ensure(deadline) returns (uint[] memory amounts){
        ...

        _swap(amounts, path, to);
    }

function swapTokensForExactTokens(
        uint amountOut,
        uint amountInMax,
        address[] calldata path,
        address to,
        uint deadline
    ) external virtual override ensure(deadline) returns (uint[] memory amounts){
        ...

        _swap(amounts, path, to);
    }
```

**Inspect:** 通过上面两个 function 我们发现，交易并不是 1 对 1 对，而是 1->1->1..->1,我们可以传[WETH, USDC, DAI]。这样最终结果还是 WETH -> DAI。出现这种场景是因为没有 WETH/DAI 的 pool，所以只能“绕远路”。  
最后我们观察`_swap(...)`这个函数，入参的 path 和 to 都是有的，amounts 是哪里来的？
答案我省略了 hhh，这个 amounts 其实就是 path[i]->path[i+1]的钱(正向，也就是 exactTokenforToken) ||path[i]<-path[i+1]的钱(逆向，也就是 TokenforExactToken),这其中的钱 uniswap 会计算好。

</details>
</details>

<details>

<summary>💹_swap in Router</summary>

```js
function _swap(uint[] memory amounts, address[] memory path, address _to) internal virtual {
        for (uint i; i < path.length - 1; i++) {
            // index i ->path[i]::path[i+1]
            // input = path[i], output = path[i + 1]
            (address input, address output) = (path[i], path[i + 1]);
            // in Uniswap Pair, the smaller address of token will be regarded as token0
            (address token0, ) = UniswapV2Library.sortTokens(input, output);
            // token0 is the smaller address of the two tokens
            uint amountOut = amounts[i + 1];
            // if input is the smaller one, then inputOutcom e is 0; if input != the smaller one, thus the logic is token1 swap token0
            (uint amount0Out, uint amount1Out) = input == token0 ? (uint(0), amountOut) : (amountOut, uint(0));
            // next pair or receiver address
            address to = i < path.length - 2 ? UniswapV2Library.pairFor(factory, output, path[i + 2]) : _to;
            IUniswapV2Pair(UniswapV2Library.pairFor(factory, input, output)).swap(
                amount0Out,
                amount1Out,
                to,
                new bytes(0)
            );
        }
    }
```

**Inspect:** `UniswapV2RouterV2::_swap`这个循环调用 v2-core 的 swap 函数，这也是为什么说`periphery`面向用户，核心的状态改变都在`v22-core`中

</details>
</details>

<details>

<summary>💹swap in Pair</summary>

```js
function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
        require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');

        //_reserve0 = X0, _reserve1 = Y0
        (uint112 _reserve0, uint112 _reserve1, ) = getReserves(); // gas savings
        //amount0Out = token0 output
        //amount1Out = token1 output
        require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');

        uint balance0;
        uint balance1;
        //scope for _token{0,1}, avoids `stack too deep` errors
        {
            address _token0 = token0;
            address _token1 = token1;
            require(to != _token0 && to != _token1, 'UniswapV2: INVALID_TO');
            if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
            if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
            if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
            balance0 = IERC20(_token0).balanceOf(address(this));
            balance1 = IERC20(_token1).balanceOf(address(this));
        }
        //      In         after  >   before  -  transfer  ?  after   - (   before - transfer  ) : 0
        //`after > before - transfer` means the token0Pool received tokens so the input is not 0
        uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');
        {
            // scope for reserve{0,1}Adjusted, avoids stack too deep errors
            uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
            uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
            require(
                // Invariant x*y=L^2
                balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000 ** 2),
                'UniswapV2: K'
            );
        }

        _update(balance0, balance1, _reserve0, _reserve1);
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }
```

**Inspect:** 在一个 Pair 的合约实例中，两个 token 需要按照 address 的地址排序，排序成 token0 和 token1。入参的 amount0Out 和 amount1Out 有一个是 0，函数的逻辑会计算 amount1In 和 amount0In。在 swap 的过程中会收取 0.3%的手续费

</details>
</details>

由此,swap 的逻辑简单梳理了一遍

### [UNIV2-3] TWAP (time weight everage price) in UniswapV2

**Discription:** 在使用 Uniswap 这种链上 Oracle 最为 price 来源的时候，很容易(100%)会受到攻击，原因就在于 Uniswap 的价格太好操控了，任何一个人做 FlashLoan 就可以让价格波动很大。由此 Uniswap 提供`TWAP`(time weight everage price)来防止价格波动。注意，TWAP 价格和现货价格是两个东西。

**Math:**

- **Spot Price 现货价格(AMM)**
  - Token X 以 Token Y 计价的现货价格:
    $$
    P\_{X/Y} = \frac{Y}{X}
    $$

- **TWAP 价格**
  - Token X 在时间区间 i 到 k 的时间加权平均价格?
    $$
    \text{TWAP}_X(T_k, T_n) = \frac{\sum\limits_{i=k}^{n-1} \Delta T_i \, P_i}{T_n - T_k}
    $$

<details>
<summary>💹 _update in pair</summary>

```js
function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
        require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');
        uint32 blockTimestamp = uint32(block.timestamp % 2 ** 32);
        uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
        if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
            // * never overflows, and + overflow is desired
            price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
            price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
        }
        reserve0 = uint112(balance0);
        reserve1 = uint112(balance1);
        blockTimestampLast = blockTimestamp;
        emit Sync(reserve0, reserve1);
    }
```

 </details>

<details>
<summary>💹 How to use TWAP in your dapp</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity >=0.4 <0.9;

import {IUniswapV2Pair} from "../../../src/interfaces/uniswap-v2/IUniswapV2Pair.sol";
import {FixedPoint} from "../../../src/uniswap-v2/FixedPoint.sol";

// Modified from https://github.com/Uniswap/v2-periphery/blob/master/contracts/examples/ExampleOracleSimple.sol
// Do not use this contract in production
contract UniswapV2Twap {
    using FixedPoint for *;

    // Minimum wait time in seconds before the function update can be called again
    // TWAP of time > MIN_WAIT
    uint256 private constant MIN_WAIT = 300;

    IUniswapV2Pair public immutable pair;
    address public immutable token0;
    address public immutable token1;

    // Cumulative prices are uq112x112 price * seconds
    uint256 public price0CumulativeLast;
    uint256 public price1CumulativeLast;
    // Last timestamp the cumulative prices were updated
    uint32 public updatedAt;

    // TWAP of token0 and token1
    // range: [0, 2**112 - 1]
    // resolution: 1 / 2**112
    // TWAP of token0 in terms of token1
    FixedPoint.uq112x112 public price0Avg;
    // TWAP of token1 in terms of token0
    FixedPoint.uq112x112 public price1Avg;

    // Exercise 1
    constructor(address _pair) {
        // 1. Set pair contract from constructor input
        pair = IUniswapV2Pair(_pair);
        // 2. Set token0 and token1 from pair contract
        token0 = pair.token0();
        token1 = pair.token1();
        // 3. Store price0CumulativeLast and price1CumulativeLast from pair contract
        price0CumulativeLast = pair.price0CumulativeLast();
        price1CumulativeLast = pair.price1CumulativeLast();
        // 4. Call pair.getReserve to get last timestamp the reserves were updated
        (, , updatedAt) = pair.getReserves();
        //    and store it into the state variable updatedAt
    }

    // Exercise 2
    // Calculates cumulative prices up to current timestamp
    //@note 这个函数计算并返回截止到当前时间戳的累积价格，用于后续计算时间加权平均价格。
    function _getCurrentCumulativePrices()
        internal
        view
        returns (uint256 price0Cumulative, uint256 price1Cumulative)
    {
        // 1. Get latest cumulative prices from the pair contract
        price0Cumulative = pair.price0CumulativeLast();
        price1Cumulative = pair.price1CumulativeLast();
        // If current block timestamp > last timestamp reserves were updated,
        // calculate cumulative prices until current time.
        // Otherwise return latest cumulative prices retrieved from the pair contract.

        // 2. Get reserves and last timestamp the reserves were updated from
        //    the pair contract
        (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast) = pair
            .getReserves();

        // 3. Cast block.timestamp to uint32, and update the timestamp of the last update
        uint32 blockTimestamp = uint32(block.timestamp);
        if (blockTimestampLast != blockTimestamp) {
            // 4. Calculate elapsed time
            uint32 dt = blockTimestamp - blockTimestampLast;

            // Addition overflow is desired
            unchecked {
                // 5. Add spot price * elapsed time to cumulative prices.
                //    - Use FixedPoint.fraction to calculate spot price.
                //    - FixedPoint.fraction returns UQ112x112, so cast it into uint256.
                //    - Multiply spot price by time elapsed
                price0Cumulative +=
                    uint256(FixedPoint.fraction(reserve1, reserve0)._x) *
                    dt;
                price1Cumulative +=
                    uint256(FixedPoint.fraction(reserve0, reserve1)._x) *
                    dt;
            }
        }
    }

    // Exercise 3
    // Updates cumulative prices
    function update() external {
        // 1. Cast block.timestamp to uint32
        uint32 blockTimestamp = uint32(block.timestamp);
        // 2. Calculate elapsed time since last time cumulative prices were
        //    updated in this contract
        uint32 dt = blockTimestamp - updatedAt;
        // 3. Require time elapsed >= MIN_WAIT
        require(dt >= MIN_WAIT, "InsufficientTimeElapsed");

        // 4. Call the internal function _getCurrentCumulativePrices to get
        //    current cumulative prices
        (
            uint256 price0Cumulative,
            uint256 price1Cumulative
        ) = _getCurrentCumulativePrices();

        // Overflow is desired, casting never truncates
        // https://docs.uniswap.org/contracts/v2/guides/smart-contract-integration/building-an-oracle
        // Subtracting between two cumulative price values will result in
        // a number that fits within the range of uint256 as long as the
        // observations are made for periods of max 2^32 seconds, or ~136 years
        unchecked {
            // 5. Calculate TWAP price0Avg and price1Avg
            //    - TWAP = (current cumulative price - last cumulative price) / dt
            //    - Cast TWAP into uint224 and then into FixedPoint.uq112x112
            price0Avg = FixedPoint.uq112x112(
                uint224(price0Cumulative - price0CumulativeLast) / dt
            );
            price1Avg = FixedPoint.uq112x112(
                uint224(price1Cumulative - price1CumulativeLast) / dt
            );
        }

        // 6. Update state variables price0CumulativeLast, price1CumulativeLast and updatedAt
        price0CumulativeLast = price0Cumulative;
        price1CumulativeLast = price1Cumulative;
        updatedAt = blockTimestamp;
    }

    // Exercise 4
    // Returns the amount out corresponding to the amount in for a given token
    function consult(
        address tokenIn,
        uint256 amountIn
    ) external view returns (uint256 amountOut) {
        // 1. Require tokenIn is either token0 or token1
        require(tokenIn == token0 || tokenIn == token1, "InvalidToken");
        // 2. Calculate amountOut
        //    - amountOut = TWAP of tokenIn * amountIn
        //    - Use FixePoint.mul to multiply TWAP of tokenIn with amountIn
        //    - FixedPoint.mul returns uq144x112, use FixedPoint.decode144 to return uint144
        if (tokenIn == token0) {
            // Example
            //   token0 = WETH
            //   token1 = USDC
            //   price0Avg = avg price of WETH in terms of USDC = 2000 USDC / 1 WETH
            //   tokenIn = WETH
            //   amountIn = 2
            //   amountOut = price0Avg * amountIn = 4000 USDC
            amountOut = FixedPoint.mul(price0Avg, amountIn).decode144();
        } else {
            amountOut = FixedPoint.mul(price1Avg, amountIn).decode144();
        }
    }
}
```

</details>

## Uniswap V3

### [UNIV3-1] Introduction of Uniswap V3

**Discription:** 对于 UniswapV2，所有的流动性都集中在一个 Pair 中，AMM 方程如下

$$
P_{X/Y} = \frac{Y}{X}
$$

只有当 L^2 足够大，这个 Pair 才可以说够“坚固”，不然很容易被恶意操控。既然如此，Uniswap V3 的出发点就是能否直接通过很小的资金来代表很大的流动性。于是 Uniswap V3 的核心`Concentrated Liquidity`就诞生了，其目的就是允许一对 Pair 可以在指定价格区间进行 swap，换句话说就是将流动性集中在指定价格区间。只有在指定价格区间的流动性才是真流动性`real reserves`,非区间的流动性都是虚拟流动性`virtual reserves`

**Diff between V2 and V3:** 在 V2 中，我们通过 X 和 Y 的存量来计算流动性和 Price，但是在 V3 中我们通过流动性和 Price 来计算 X 和 Y。Active Liqituity 用 ERC712 来表示，而不是 ERC20。Swap Fee 也存在区别，V2 是固定 0.3%，但是 V3 有四种不同的计费规则。TWAP 的计算也会有区别

![UniswapV3OverView](./image/uniswapV3OverView.png)

### [UNI-V3-2] Spot Price

**Discription:** 在 UniswapV3 中如何计算 SpotPrice 现货价格？

<details>
<summary>✅SLOT0 in Pair</summary>

```js
struct Slot0 {
        // the current price
        uint160 sqrtPriceX96;
        // the current tick
        int24 tick;
        // the most-recently updated index of the observations array
        uint16 observationIndex;
        // the current maximum number of observations that are being stored
        uint16 observationCardinality;
        // the next maximum number of observations to store, triggered in observations.write
        uint16 observationCardinalityNext;
        // the current protocol fee as a percentage of the swap fee taken on withdrawal
        // represented as an integer denominator (1/x)%
        uint8 feeProtocol;
        // whether the pool is locked
        bool unlocked;
    }
```

</details>

注意到 Slot0 中有两个参数`sqrtPriceX96`和`tick`。 任意知道这两个变量的其中一个就可以计算出 Price。

<details>
<summary>✅Calculate Price By Tick</summary>

**Discription:** 用 Tick 表示 Price 也叫离散价格模型。在 Uniswap V3 中，tick 是一个 整数，价格(token1 / token0)按固定比例离散化：

$$
\text{price}(tick) = 1.0001^{\,tick}
$$

其中

- $\text{price}$ 表示 token1 / token0
- $tick \in \mathbb{Z}$

每一个 tick 约等于`0.01%`的价格变化

假设我们已经知道了 WETH/USDT[^weth/usdt_pool]池的 tick

[^weth/usdt_pool]: https://etherscan.io/address/0x4e68ccd3e89f51c3074ca5072bbac773960dfa36#readContract

通过 etherscan 可以看到如下信息

```txt
The 0th storage slot in the pool stores many values, and is exposed as a single method to save gas when accessed externally.

 sqrtPriceX96 uint160, tick int24, observationIndex uint16, observationCardinality uint16, observationCardinalityNext uint16, feeProtocol uint8, unlocked bool

[ slot0 method Response ]
  sqrtPriceX96   uint160 :  4586418891309846984317130
  tick   int24 :  -195150
  observationIndex   uint16 :  55
  observationCardinality   uint16 :  150
  observationCardinalityNext   uint16 :  150
  feeProtocol   uint8 :  102
  unlocked   bool :  true
```

```python
# Weth = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2
# USDT = 0xdAC17F958D2ee523a2206206994597C13D831ec7
# Weth is token0 and USDT is token1
tick = -194624

# p repercent token0 in terms of token1 (1 weth = p usdt)
#      Raw_Amount_USDT(token1)
# p = ----------------------
#      Raw_Amount_WETH(token0)

p = 1.0001 ** tick

decimal_usdt = 6
decimal_weth = 18

price = p * (10 ** (decial_weth - decimal_usdt))

```

换一个例子，假设我们在 Weth/UDSC 池子里

```python

# Weth = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2
# USDC = 0xA0...
# USDC is token0 and WETH is token1
tick = 194609

# p repercent token0 in terms of token1 (1 udsc = p weth)
#      Raw_Amount_WETH(token1)
# p = ----------------------
#      Raw_Amount_USDC(token0)

p = 1.0001 ** tick

decimal_weth = 18
decimal_usdc = 6

#          raw_amount_token1          decimal_token0(1e6)
# price = ----------------------- X --------------------------
#          raw_amount_token0          decimal_token1(1e18)
price = p * (10 ** (decial_usdc - decimal_weth))

price2 = 1/price
```

</details>

$$
\text{price}(tick) = 1.0001^{\,tick}
$$

- tick < 0 : 代表着 token1 更少更稀有
- tick > 0 : 代表着 token0 更少更稀有

<details>
<summary>✅Calculate Price By SqrtPriceX96</summary>

**Discription:** 在 Uniswap V3 中，池子的核心价格状态不是直接存 price，而是存：

$$
\text{sqrtPriceX96}
=
\sqrt{\frac{A_1}{A_0}}
\cdot
2^{96}
$$

其中：

现在有 WETH / USDT pool

```python

sqrt_q_96  = 4586418891309846984317130
Q96 = 2 ** 96

p = (sqrt_q_96/Q96) ** 2

price = p / 1e6 * 1e18

```

</details>

另一个需要知道的是`tick`和`sqrt_q_96`的相互转化

<details>
<summary>💹SqrtQ96 To Tick</summary>

$$
P
=
1.0001^{\text{tick}}
=
\left(
\frac{\text{sqrtPriceX96}}{Q96}
\right)^2
$$

$$
\text{tick}
=
\frac{
2 \cdot \log\!\left(\frac{\text{sqrtPriceX96}}{Q96}\right)
}{
\log(1.0001)
}
$$

```python
Q96 = 2 ** 96
sqrt_p_x96 = 1386025840740905446350612632896904
tick = 195402

t = 2 * math.log(sqrt_p_x96 / Q96) / math.log(1.0001)
```

结果发现整数位都是一样的，最后取整数位 tick 相同

</details>

<details>
<summary>✅Exersice</summary>

FullMath[^github-fullmath]是 UniswapV3 中用来安全计算的库。

[^github-fullmath]: https://github.com/Uniswap/v3-core/blob/0.8/contracts/libraries/FullMath.sol

```js
pragma solidity 0.8.24;

import {Test, console2} from "forge-std/Test.sol";
import {IUniswapV3Pool} from "../../../src/interfaces/uniswap-v3/IUniswapV3Pool.sol";
import {UNISWAP_V3_POOL_USDC_WETH_500} from "../../../src/Constants.sol";
import {FullMath} from "../../../src/uniswap-v3/FullMath.sol";

contract UniswapV3SwapTest is Test {
    // token0 (X)
    uint256 private constant USDC_DECIMALS = 1e6;
    // token1 (Y)
    uint256 private constant WETH_DECIMALS = 1e18;
    // 1 << 96 = 2 ** 96
    uint256 private constant Q96 = 1 << 96;
    IUniswapV3Pool private immutable pool =
        IUniswapV3Pool(UNISWAP_V3_POOL_USDC_WETH_500);

    // Exercise 1
    // - Get price of WETH in terms of USDC and return price with 18 decimals
    function test_spot_price_from_sqrtPriceX96() public {
        //@note p = amount_weth/amount_usdc = usdc in terms of weth
        //@note p has 18-6 = 12 decimals
        //@note 1/p has 6-18 = -12 decimals
        uint256 price = 0;
        IUniswapV3Pool.Slot0 memory slot0 = pool.slot0();

        // Write your code here
        // Don’t change any other code

        // sqrtPriceX96 * sqrtPriceX96 might overflow
        // So use FullMath.mulDiv to do uint256 * uint256 / uint256 without overflow

        //@note sqrtPriceX96^2 = (sqrt(p) * Q96)^2
        //@note                = p * Q96^2
        //@note                = p * 2 ** 192
        //也就是说我们还有64bit用来保存p 但是如果p的decimal时18的时候，p最多只能到达18
        //由此上面的方法是错误的
        //@note sqrtPriceX96^2 / Q96^2 = sqrt(p)^2
        //@note                        = p
        //这个方法当sqrtPriceX96很小的话，会导致p = 0
        //@note 因此，我们需要用FullMath.mulDiv来计算

        price = FullMath.mulDiv(slot0.sqrtPriceX96, slot0.sqrtPriceX96, Q96);
        // price = p * Q96
        // 1/price = 1 / (p * Q96) 在solidity中，1/一个很大的数 -->结果为0，向下取整
        price = (Q96 * 1e12 * 1e18) / price;

        assertGt(price, 0, "price = 0");
        console2.log("price %e", price);

        /*
        [PASS] test_spot_price_from_sqrtPriceX96() (gas: 14755)
        Logs:
        price 2.3744513783461654702155066169421839144e37k
        */
    }
}

```

</details>

# [Non-technical]

### [N-1] How to build My own IP

[^来源于]: https://www.bilibili.com/video/BV1zmQiYME3F?spm_id_from=333.788.recommend_more_video.2&trackid=web_related_0.router-related-2206146-hhltv.1768275068198.532&vd_source=3831eae2cf36582eee1ac4a48490adea

### [N-2] Phased Plan from 1/12 to 2/8 in 2026

**Discription: **我的目标岗位是合约审计。这个岗位门槛真的很高，而且是和合约开发高度耦合的。一个优秀的审计员无疑也是一个优秀的开发者，二者需要的知识储备，技术栈都是相似的。

这四周我需要学习市面上主流 Defi 的源码，学习里面优秀的设计模式，以及其他 Dapp 如何接入这些 Defi protocal

- uniswap V3
- uniswap V4
- Aave V4
- GMX

一周学一个吧，前两周允许我进度慢一点，还在期末考试中。除此之外继续跟进实习计划的任务，目标是保持在排行榜的前面，也希望能在实习结束接到一个开发的实习。目前我 Solidity 的代码量还是太少了，还是没能做到随心所欲的地步，需要一些开发来逐渐精通。

这一段时间应该是参加不了 Competitve Audit 了，不过这些比赛我今后长期活跃在其中。还有黑客松，我也会经常参加。最近还觉得需要在社交媒体中建立自己的权威，比如在推特上，这个方向我也会关注，不过不是现在这四周的重心。

我能感觉到现在是我人生的一个小小新阶段的起步。未来会专注于合约审计和开发并行学习，我觉得自己还是需要开发的经验作为积累，二者并行的学习模式可能是最适合现阶段的我的。

### [N-3] Colearning in 2026/01/15

**Discription:** 需要在社交媒体中传播价值，也许我可以尝试发布一些东西，打造一个个人 IP，这需要运营

出发点: 说服别人为什么要用这个产品(为什么要加入这个社区),从增长的角度出发。

总结就是增长和留存
