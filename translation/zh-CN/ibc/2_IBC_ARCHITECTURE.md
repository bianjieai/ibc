# 2：区块链间通信协议架构

**这是 IBC 协议的高层次架构和数据流的概述。**

**有关 IBC 规范中使用的术语的定义，请参见[此处](./1_IBC_TERMINOLOGY.md) 。**

**有关广泛的协议设计原则，请参见[此处](./3_IBC_DESIGN_PRINCIPLES.md) 。**

**有关一组示例用例，请参见[此处](./4_IBC_USECASES.md) 。**

**有关设计模式的讨论，请参见[此处](./5_IBC_DESIGN_PATTERNS.md) 。**

本文档概述了区块链间通信协议（IBC）协议栈中身份认证，传输和排序层的大致架构。本文档没有描述特定的协议详细信息-它们包含在各个 ICS 中。

> 注意： *账本* ， *链*和*区块链*根据其通俗用法在本文档中可互换使用。

## 什么是 IBC？

*区块链间通信协议*是一种可靠且安全的模块间通信协议，其中模块是在独立机器上运行的确定性进程，包括多副本状态机（例如“区块链”或“分布式账本”）。

任何基于可靠且安全的模块间通信的应用程序都可以使用 IBC。示例应用程序包括跨链资产转移，原子交换，多链智能合约（不论是否拥有相互可理解的虚拟机）以及各种数据和代码分片应用。

## IBC 不是什么？

IBC 不是应用层协议：它仅处理数据传输，身份认证和可靠性。

IBC 不是原子交换协议：它支持任意跨链数据传输和计算。

IBC 不是通证转移协议：通证转移是 IBC 协议上的一种可能的应用层场景。

IBC 不是分片协议：不是将单个状态机拆分成链，而是在不同链上的不同状态机共享一些公共接口。

IBC 不是二层扩容协议：所有实现 IBC 的链都存在于同一“层”上，尽管它们可能位据网络拓扑中的不同位置，但唯一的根链或唯一的验证人集合不是必须的。

## 动机

在撰写本文时，两个主要的区块链，比特币和以太坊目前分别支持每秒约 7 和 20 笔交易。尽管主要用户群还是那些早期采用者，两条链在最近的几年中一直处于满负荷状态运行。吞吐量是大多数区块链使用场景的限制，而吞吐量根本上受限于分布式状态机，因为网络中的每个（验证）节点都必须处理每笔交易（抛开未来基于零知识的结构，这目前超出了范围） ，存储所有状态，并与其他验证节点通信。更快的共识算法（例如 [Tendermint](https://github.com/tendermint/tendermint) ）或许可以将吞吐量提高一个很大的常数因子，但是由于这个原因仍无法无限扩展。为了支持促进分布式账本应用程序广泛部署所需的交易吞吐量，多样性和成本效率，必须同时运行多个独立的共识实例，并分离之间的执行和存储。

一种设计方向被称作“分片”，是切片单个可编程状态机为不同的链，这些链同时执行并存储状态的不相交分区。为了探讨安全性和活性，并正确的在分片之间路由数据和代码，这个设计必须采取“自上而下的方法” —构建特定的网络拓扑，特征是单个根账本和一个星状或树状分片结构，并用工程协议规则和激励措施强制实施该拓扑。这种方法在简单性和可预测性方面具有优势，但面临着严峻的[技术](https://medium.com/nearprotocol/the-authoritative-guide-to-blockchain-sharding-part-1-1b53ed31e060) [问题](https://medium.com/nearprotocol/unsolved-problems-in-blockchain-sharding-2327d6517f43) ，要求所有分片都遵循单个验证人集合（或随机选择的子集）和单个状态机或相互可理解的虚拟机，并且可能会面临未来的社会可扩展性问题，因为有必要就网络拓扑的变更达成全球共识。

此外，任何单一的共识算法，状态机和抗女巫攻击个体都可能无法提供必要的安全性和多样性。共识实例受其支持的独立运营者数量的限制，这意味着，破坏特定运营者所获得的平摊收益会随着共识实例保有的价值增加而增加，而破坏运营者的成本则始终依照最经济的方式（例如物理密钥泄露或社会工程学），这几乎不会无限增加。单个全局状态机必须迎合多样化应用程序集的共同点，因此与专用状态机相比，它不太适合任何特定应用程序。单个共识实例的运营者可能会滥用其特权地位，从无法轻易选择退出的应用程序中提取租金。最好构造一种机制，通过该机制，独立的主权共识实例和状态机可以安全，自愿的交互，同时仅共享最小化的必需的公共接口。

*区块链间通信协议*对扩容和互操作性问题的不同表示形式采用了不同的方法：实现以未知拓扑排列的异构分布式账本网络的安全，可靠的互操作，在可能的情况下保留保密性，使账本可以多样化，彼此独立或独立于特定的拓扑或状态机设计进行开发和重组。在广泛的，动态的互操作链网络中，预期会出现零星的拜占庭错误，因此该协议还必须根据所涉及的应用程序和账本的要求来检测，缓解并包含拜占庭错误的潜在损害。有关设计原则的详细列表，请参见[此处](./3_IBC_DESIGN_PRINCIPLES.md) 。

为了促进这种异构互操作，区块间链通信协议采用“自下而上”的方法，指定实现两个账本之间的互操作所必需的一组要求，函数和属性，然后指定可能构成多个互操作账本的不同方式，账本保留更高级别协议的要求，并在安全/速度折衷空间中占据不同的位置。因此，IBC 不做任何假设，也不需要知道整个网络拓扑和账本的实现，只需要知道的最小函数集和属性即可。实际上，在 IBC 中账本被定义为轻客户端的一些共识验证函数，从而扩展了“账本”的含义，包括了单一机器和类似的复杂共识算法。

IBC 是一种端到端，面向连接的状态协议，用于在独立计算机上的模块之间进行可靠的，有序的（可选）、带身份认证的通信。预计 IBC 实现将与主机状态机上的更高级别的模块和协议共存。承载 IBC 的状态机必须提供一组特定的功能来验证共识记录和生成加密承诺证明，并且 IBC 数据包中继器（链下进程）应能够访问网络协议和物理数据链接，以读取一台机器的状态，然后将数据提交给另一台机器。

## 范围

IBC 处理在不同机器的模块之间被中继的结构化数据包的身份认证、传输和排序工作。该协议是在两台机器上的模块之间定义的，但设计可以安全的同时用于以任意拓扑连接的任意数量的机器上的任意数量的模块之间。

## 接口

IBC 位于两类模块之间，一类是智能合约，其他状态机组件或状态机上其他独立的应用程序逻辑部分，另一类是基础共识协议，机器和网络基础结构（例如 TCP/IP）。

IBC 为模块提供了一组函数，类似于为与同一状态机上的另一个模块进行交互的模块可能提供的函数：在已建立的连接和通道上发送数据包和接收数据包（用于身份认证和排序的原语，请参阅[定义](./1_IBC_TERMINOLOGY.md) ）-除了用于管理协议状态的调用之外：创建和关闭连接/通道，选择连接/通道，数据包传递选项以及检视连接和通道状态。

IBC 假定 [ICS 2](../../spec/ics-002-client-semantics) 中定义的底层共识协议和机器的功能和特性，主要是最终性（或阈值最终性小工具），可廉价验证的共识记录和简单的键/值存储功能。在网络方面，IBC 仅需要最终的数据传递-不假定身份认证，同步或有序属性（这些属性将在稍后精确定义）。

### 协议关系

```
+------------------------------+                           +------------------------------+
| Distributed Ledger A         |                           | Distributed Ledger B         |
|                              |                           |                              |
| +--------------------------+ |                           | +--------------------------+ |
| | State Machine            | |                           | | State Machine            | |
| |                          | |                           | |                          | |
| | +----------+     +-----+ | |        +---------+        | | +-----+     +----------+ | |
| | | Module A | <-> | IBC | | | <----> | Relayer | <----> | | | IBC | <-> | Module B | | |
| | +----------+     +-----+ | |        +---------+        | | +-----+     +----------+ | |
| +--------------------------+ |                           | +--------------------------+ |
+------------------------------+                           +------------------------------+
```

## 运作方式

IBC 的主要目的是在独立主机上运行的模块之间提供可靠的，经过身份认证的有序通信。这需要以下领域的协议逻辑：

- 数据中继
- 数据保密性和易读性
- 可靠性
- 流控
- 身份认证
- 富状态性
- 多路复用
- 序列化

以下各段概述了 IBC 中每个领域的协议逻辑。

### 数据中继

在 IBC 体系结构中，模块不是通过网络基础结构直接向彼此发送消息，而是创建要发送的消息，然后通过负责监视的“中继器进程”对消息进行物理中继。 IBC 假定存在一组中继器进程，这些中继器进程可以访问基础网络协议堆栈（可能是 TCP/IP，UDP/IP 或 QUIC/IP）和物理互连基础设施。这些中继器进程监视一组实现 IBC 协议的机器，连续扫描每台机器的状态，当有传出数据包提交时，中继器进程就到另一台机器上执行交易。为了正常操作和处理在两台机器之间的连接，IBC 仅要求至少存在一个可以在机器之间进行中继的正确且活跃的中继器进程。

### 数据保密性和易读性

IBC 协议仅要求可以正确操作 IBC 协议所需的最少数据，使协议可用且易读（以标准格式序列化），并且状态机可以选择使该数据仅对特定中继器可用（但这里的细节超出本规范的范围）。这些数据包括共识状态，客户端，连接，通道和数据包信息，以及构造状态中特定键/值对的包含或不包含证明所必需的任何辅助状态结构。所有必须证明给另一台机器的数据也必须易读；即必须以本规范定义的格式序列化。

### 可靠性

网络层和中继器进程可能以任意方式运行，丢弃，乱序或发送重复数据包，故意尝试发送非法交易或进行其他拜占庭错误方式的操作。这必须不能损害 IBC 的安全性和活性。可靠性是通过为 IBC 连接（在发送时）发送的每个数据包分配一个序号来实现的，该序号由接收机上的 IBC 处理模块（实现 IBC 协议的状态机部分）检查并提供一种发送机在发送更多数据包或采取进一步措施之前检查接收计算机实际上是否已接收并处理了数据包的方法。加密承诺用于防止数据报伪造：发送方机器承诺传出数据包，而接收方计算机检查这些加密承诺，因此被中继器在传输过程中更改的数据报将被拒绝。IBC 还支持无序通道，该通道不强制要求数据包的接收符合发送的数据包顺序，但仍强制执行仅一次送达。

### 流控

IBC 没有提供计算层面或经济层面的流控规定。底层机器将具有自己的计算吞吐量限制和流控机制（例如“gas 费用”市场）。应用程序层面的经济流控（根据其内容限制特定数据包的速率）可能对确保安全性（限制一台机器上的值）以及抑制拜占庭行为造成的损害（允许在挑战期内提交矛盾行为证明，然后关闭连接）有用。例如，通过 IBC 通道转移价值的应用程序可能希望限制每个块的价值转移率，以限制潜在的拜占庭行为所造成的损害。IBC 为模块提供了拒绝数据包的功能，并将细节留给了更高级别的应用程序协议。

### 身份认证

IBC 中的所有数据报都经过了身份认证：由发送机的共识算法敲定的块必须给发送出的数据包提供加密承诺，并且接收链的 IBC 处理模块必须验证数据包被发送的共识记录和加密承诺证明，之后才可以使用它。

### 富状态性

如上所述，可靠性，流控和身份认证要求 IBC 初始化并维护每个数据流的某些状态信息。此信息分为两个抽象：连接和通道。每个连接对象都包含有关已连接计算机的共识状态的信息。特定于一对模块的每个通道都包含有关协商的编码和多路复用选项以及状态和序号的信息。当两个模块希望进行通信时，它们必须在其两台机器之间找到一个现有的连接和通道，如果不存在则初始化一个新的连接和通道。初始化连接和通道需要多步握手，一旦完成，就确保了在这个连接下仅连接了两个目标机器，两个模块，并对将来的数据报进行身份认证，编码，对于通道，还需要保证排序。

### 多路复用

为了允许单个主机中的多个模块同时使用 IBC 连接，IBC 在每个连接中提供了一组通道，每个通道的数据流拥有唯一标识，所以数据包可以按顺序发送（在有序的情况下） ，并且接收方机器上始终保持仅一次送达。通常希望通道与每台机器上的单个模块关联，但是一对多和多对一的通道也是可能的。通道的数量是无限的，这有助于并发吞吐量仅受限于底层计算机的吞吐量，而且仅需一个连接即可跟踪共识信息（该连接的所有通道分摊了共识记录验证成本）。

### 序列化

IBC 充当了彼此无法理解的机器之间的接口边界，并且必须提供必要的最少的数据结构编码和数据报格式集，以允许两台都正确实现协议的机器相互理解。为此，IBC 规范在此仓库中定义了以 proto3 格式提供了在通过 IBC 进行通信的两台机器之间进行序列化，中继或证明检查的数据结构的典范编码。

> 注意，proto3 的一个子集提供了典范的编码（相同的结构始终序列化为相同的字节）。因此，禁止使用 map 和未知字段。

## 数据流

IBC 可以概念化为一个分层协议栈，数据从上到下（在发送 IBC 数据包时）或自下而上（在接收 IBC 数据包时）经过协议栈。

“处理程序”是状态机中实现 IBC 协议的一部分，该状态机负责将模块或数据包的调用进行转换，并适当的路由这些调用到通道和连接。

考虑两条链 *A* 和 *B* 之间的 IBC 数据包的路径：

### 图表

```
+---------------------------------------------------------------------------------------------+
| Distributed Ledger A                                                                        |
|                                                                                             |
| +----------+     +----------------------------------------------------------+               |
| |          |     | IBC Module                                               |               |
| | Module A | --> |                                                          | --> Consensus |
| |          |     | Handler --> Packet --> Channel --> Connection --> Client |               |
| +----------+     +----------------------------------------------------------+               |
+---------------------------------------------------------------------------------------------+

    +---------+
==> | Relayer | ==>
    +---------+

+--------------------------------------------------------------------------------------------+
| Distributed Ledger B                                                                       |
|                                                                                            |
|               +---------------------------------------------------------+     +----------+ |
|               | IBC Module                                              |     |          | |
| Consensus --> |                                                         | --> | Module B | |
|               | Client -> Connection --> Channel --> Packet --> Handler |     |          | |
|               +---------------------------------------------------------+     +----------+ |
+--------------------------------------------------------------------------------------------+
```

### 步骤

1. 在链 *A* 上
    1. 模块（取决于应用）
    2. 处理程序（在不同的 ICS 中定义）
    3. 数据包（在 [ICS 4](../spec/ics-004-channel-and-packet-semantics) 中定义）
    4. 通道（在 [ICS 4](../spec/ics-004-channel-and-packet-semantics) 中定义）
    5. 连接（在 [ICS 3](../spec/ics-003-connection-semantics) 中定义）
    6. 客户端（在 [ICS 2](../spec/ics-002-client-semantics) 中定义）
    7. 共识（与传出数据包确认交易）
2. 链下
    1. 中继器（在 [ICS 18](../spec/ics-018-relayer-algorithms) 中定义）
3. 在链 *B* 上
    1. 共识（确认传入数据包的交易）
    2. 客户端（在 [ICS 2](/../spec/ics-002-client-semantics) 中定义）
    3. 连接（在 [ICS 3](/../spec/ics-003-connection-semantics) 中定义）
    4. 通道（在 [ICS 4](/../spec/ics-004-channel-and-packet-semantics) 中定义）
    5. 数据包（在 [ICS 4](/../spec/ics-004-channel-and-packet-semantics) 中定义）
    6. 处理程序（在不同的 ICS 中定义）
    7. 模块（取决于应用）