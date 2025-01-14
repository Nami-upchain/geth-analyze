

# Oracle

oracle 翻译是预言机，英文中的意思是预卜先知，知晓消息的意思。在区块链里用于合约获取链外的数据。例如你想把比特币转换成美元，如果在链上进行，那么就需要从链外获取比特币和美元的汇率，例如[price feed oracles](https://developer.makerdao.com/feeds/)。但是以太坊是封闭的系统，直接与外界交互很容易破坏 EVM 安全性，因此才用了预言机作为中间层，沟通链上和链外。详细可见[chainlink的文档](https://chain.link/education/blockchain-oracles)和[官方文档](https://ethereum.org/en/developers/docs/oracles/)。

在以太坊上，**oracle 是已经部署的智能合约和链外组件，它可以查询 API 提供的信息，然后给其他合约发消息，更新合约的数据**。但是只相信唯一的数据源也是很不可靠的方式，通常是多个数据源。我们可以自己创建，也可以直接使用服务商提供的服务。

一般 oracle 机制如下：

1. 到了需要链外数据的时候，合约触发事件。
2. 链外的接口监听事件的日志。
3. 链外接口处理事务，然后交易的方式返回数据给合约。

![1_Cs3w9iFmhIfkyg3Kg_FzFw](http://blog-blockchain.xyz/202203260125659.png)

## oracle 实例

下面是一个例子，从网络导入合约库，获取接口信息，然后创建合约类型 `AggregatorV3Interface` 的变量 `priceFeed`，然后结合获取的接口信息，在构造函数里创建在特定地址已经部署好的合约实例，调用函数`priceFeed.latestRoundData()`，返回的是元组，因此用多个数据接收。这样就获得了最新的 ETH 和 USD 的汇率。而我们导入的合约`priceFeed` 以及它在链外的配套接口，被称作预言机 oracle。类似的，我们也可以通过 oracle 解决链上难以产生可靠的随机数的问题。  

​	更多的例子可以看 chainlink 这些提供商，提供的文档，详细地说明了流程。也可以看这个[教程](https://github.com/pedroduartecosta/blockchain-oracle)。

```js
// This example code is designed to quickly deploy an example contract using Remix.

pragma solidity ^0.6.7;

import "https://github.com/smartcontractkit/chainlink/blob/master/evm-contracts/src/v0.6/interfaces/AggregatorV3Interface.sol";

contract PriceConsumerV3 {

    AggregatorV3Interface internal priceFeed;

    /**
     * Network: Kovan
     * Aggregator: ETH/USD
     * Address: 0x9326BFA02ADD2366b30bacB125260Af641031331
     */
    constructor() public {
        priceFeed = AggregatorV3Interface(0x9326BFA02ADD2366b30bacB125260Af641031331);
    }

    /**
     * Returns the latest price
     */
    function getLatestPrice() public view returns (int) {
        (
            uint80 roundID, 
            int price,
            uint startedAt,
            uint timeStamp,
            uint80 answeredInRound
        ) = priceFeed.latestRoundData();
        return price;
    }
}
```

## 确保 oracle 安全的方式

### Swiss-Cheese 模型

我们采用多层的结构保证数据的可信性，只有多层结构中只要有一个正常工作，则代表 oracle 提供的数据可信。这样也避免了单一数据来源的最脆弱环节失效容易导致漏洞的问题

![1_HCQQSCkvdaUWWG3lYYH9FA](http://blog-blockchain.xyz/202203260126792.png)

### 多数据源

可以在链上采用多个数据源，那么只有绝大多数数据都失效或者oracle合约本身存在漏洞时，oracle 才会失效。

实际上，多个可信的数据来源在链上处理是比较耗费 gas 的，因此提出了通过密码学手段，在链外汇总数据，然后发给合约。

### 多个 oracle

多用几个 oracle 一起验证安全性会提高很多，但是所有 oracle 都传入不正确的数据时，也可能出问题。当智能合约有多个 oracle 来源时，选择哪一个也是需要设计合理的共识机制的。一般而言，多个 oracle 需要满足：

1. 每个 oracle 无法确认其他 oracle 的身份。这可以让他们无法串通。
2. oracle 之间无法沟通，并且不会互相影响。例如，某个 oracle 有 40% 的投票权，他无法影响其他 oracle，让他们做出相同的选择。
3. 当所有 oracle 都提供数据之前，每个 oracle 提供的数据都是无法确认的。这相当于在投票时，只有每个人都投完票之后，才公布结果。
4. oracle 都带有权重，防止有人控制大量节点，成为分布式系统中的 “大多数”。

### 利益一致

完全区中心化的 oracle 是很危险的，我们无法预见数据提供者的行为。但是，可以尝试将 oracle 融入类似于挖矿的过程，如果执行者按规定执行，则给予奖励，否则就会产生损失。

## Oracle 可能的漏洞

​	单纯创建一个点对点的去中心化系统并不难，但是保证在去中心化系统中某些必要组件的可信性，却是一个难题。

- 为了节省验证数据的计算开销，大节点可能在收集数据之后，在链外分享给它控制的节点。如果大节点收集的数据是错误的，那么拥有错误信息的节点容易占大多数，形成另类的女巫攻击。

- 恶意的 oracle 可能会抄袭别人的数据。

- 单一的 oracle的情况，如果数据有损坏，那么在链上是很难检测的。

- 区块链数据都是公开的，即使每个 oracle 的数据加密，执行过程中很难保证敏感的信息不会泄露。

  详细可参考 [Decentralised Oracles: a comprehensive overview](https://medium.com/fabric-ventures/decentralised-oracles-a-comprehensive-overview-d3168b9a8841)

## Oracle 的源码实现

从类型定义可见，checkpoint oracle 实际上是一个合约，它的方法也是和普通合约封装类似，

1.   通过地址绑定到已部署的合约，调用该合约。
2.   合约地址。

特殊的在于：

1.   检查某个状态阶段的可信点（检查点）。
2.   生成新的检查点。

附检查点的含义：oracle 的检查点，实际上是一个标记，用于确认这个状态和之前的状态是可信的。在区块链上，检查点往往是有足够的可信实体共同签名后，正式生成。它意味着检查点的状态是不可逆的，无条件可信的。这也是区块链防止造假的手段之一。