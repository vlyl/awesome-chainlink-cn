# 《精通以太坊》预言机

[本文摘自《精通以太坊》一书第11章预言机部分]

在本章中，我们将讨论预言机（oracle），它是可以为以太坊智能合约提供外部数据源的系统。 “oracle”一词来自希腊神话，代表能够与神灵交流的人，他们可以看到未来的愿景。在区块链的上下文中，预言机是一个可以回答以太坊外部问题的系统。在理想情况下，预言机是无信任的系统，这意味着它们不需要被信任，因为它们是按照去中心化的原则运行的。

## 为什么需要预言机？

以太坊平台的一个关键组件是EVM，它能够在分散网络中的任何节点上执行程序并更新受共识规则约束的以太坊状态。为了保持共识，EVM的执行过程必须完全确定，并且仅基于以太坊状态和签名交易的共享上下文。这产生了两个特别重要的后果：一个是EVM和智能合约没有内在的随机性来源；另一个是外部数据只能作为交易的数据载荷引入。

让我们进一步分析这两个后果。首先我们要理解为什么在EVM中需要禁止真正的随机函数，不让它们为智能合约提供随机性。请考虑在执行此类函数后对尝试达成共识的影响：节点A将执行命令并代表智能合约存储3在其存储中，而节点B执行相同的智能合约，将存储7。因此，节点A和节点B就结果状态应该是什么将得出不同的结论，尽管在相同的上下文中运行完全相同的代码。实际上，每次评估智能合约时，可能会达到不同的结果状态。因此，由于其众多节点在世界各地独立运行，网络将无法就结果状态应该是什么达成去中心化的共识。在实践中，它会比这个例子复杂得多，因为包括以太币转移在内的连锁效应会以指数方式增长。

注意，伪随机函数，例如加密安全哈希函数（它们是确定性的，因此实际上也是EVM的一部分），对于许多应用来说是不够的。假想一个猜硬币决胜负的游戏场景，这个游戏依赖于扔硬币正反面的随意性。矿工可以轻易破解这个游戏：他们只需要打包那些对其有利的随机结果即可。那么我们如何解决这个问题呢？既然所有节点都可以就签名交易的内容达成一致，那么我们就可以引入外部信息，包括随机性、价格信息、天气预报等，作为发送到网络的交易的数据部分。但是，这些数据本身无法信任，因为它的来源无法核实。因此，我们刚刚并没有完全解决这个问题。我们使用预言机尝试解决这些问题，在本章的其余部分将详细讨论这些问题。

## 预言机的应用场景和示例

理想情况下，预言机提供了一种无信任（或至少近乎无信任）的方式来获取外在的（即“真实世界”或“链外”）信息，例如足球比赛的结果、黄金的价格或真正的随机数字，用于以太坊平台上的智能合约。它们还可用于直接将数据安全地中继到DApp前端。因此，可以将预言机视为弥合链外世界与智能合约之间差距的机制。允许智能合约基于真实世界的事件和数据来强制执行合约关系，从而大大扩展了它们的范围。但是，这也会给以太坊的安全模型带来外部风险。考虑一个“智能遗嘱”合约——当一个人去世时分配资产。这是智能合约领域中经常讨论的内容，并突出了可信任的预言机的风险。如果由这样的合约控制的继承金额足够高，那么在所有者死亡之前攻击预言机（发出假的死讯）并触发资产分配的动机是非常强烈的。

请注意，某些预言机提供针对特定私有数据源的数据，例如学术证书或政府ID。这些数据的来源，如大学或政府部门，是完全可信的，数据的真实性是主观的（真相只能通过来源的权威来确定）。因此，不能无信地提供这样的数据，即不信任来源，因为没有独立可验证的客观事实。因此，我们将这些数据源包含在我们对预言机的定义中，因为它们还为智能合约提供了数据桥梁。它们提供的数据通常采用证明的形式，如护照或成就记录。“证言”将成为未来区块链平台成功的重要组成部分，特别是在验证身份或声誉的相关问题方面，因此探索区块链平台如何为其提供服务非常重要。

可能由预言机提供的更多数据示例包括：

- 物理随机数源或熵源（例如量子现象或热现象）：如在彩票智能合约中公平地选出获奖者。
- 与自然灾害相关的参数触发器：触发大型自然灾害债券智能合约（例如地震的里氏震级测量债券）。
- 汇率数据：例如让加密货币与法定货币精确挂钩。
- 资本市场数据：例如为一揽子代币化资产或证券定价。
- 指标引用数据：例如将利率纳入智能金融衍生品合约。
- 统计与准统计数据：安全标识、国家代号、货币代号等。
- 时间和间隔数据：基于精准的SI（国际单位制）时间度量的事件触发器。·天气数据：例如基于天气预报的保险费计算器。
- 政治事件：预测市场的走势。·运动事件：预测市场走势以及体育博彩相关的合约。·地理定位数据：例如供应链跟踪。
- 损坏程度核验：保险合约。
- 其他区块链上发生的事件：可互操作函数。
- 以太币市场价格：例如gas价格预言机。
- 航班统计数据：例如用于团体和俱乐部的机票合同。

在接下来的部分，我们将研究可以实现预言机的一些方法，包括基本的预言机的设计模式、计算性预言机、去中心化的预言机以及Solidity中的预言机客户端实现。

## 预言机的设计模式

根据定义，所有的预言机都提供了一些关键功能。这些能力包括：

从链外的数据源收集数据。

使用签名消息在链上传输数据。

将数据放入智能合约的存储空间，使数据可用。

一旦数据在智能合约的存储中可用，其他智能合约就可以通过调用预言机智能合约的“检索”功能来访问它；它也可以通过“查看”预言机的存储直接由以太坊节点或支持网络的客户端访问。

设置预言机的三种主要方式可以分为*请求与响应、发布与订阅*和*立即读取*。

让我们从最简单的“*立即读取*”式预言机开始，这种预言机提供即时决策所需的数据，例如“ethereumbook.info的地址是什么”或“这个人是否超过18岁”。那些希望查询此类数据的人倾向于在“即时”的基础上这样做；查找是在需要信息时完成的，可能永远不会再次查找。这种预言机的例子包括那些持有组织数据或由组织发布数据（例如学术证书、拨号代码、机构会员资格、机场标识符、自主ID等）的预言机。这种类型的预言机一旦将数据存储在其合约存储中，其他智能合约就可以使用对预言机合约的请求调用来查找。它可能会更新。预言机存储中的数据也可用于通过区块链启用（即，以太坊客户端连接）应用程序直接查找，而无须通过调整并产生发布交易的gas成本。想要检查买酒顾客年龄的商店可以这样使用预言机。这种类型的预言机对于可能需要运行和维护服务器来回答此类数据请求的组织或公司具有吸引力。注意：由预言机存储的数据可能不是预言机正在服务的原始数据，例如，出于效率或隐私原因，大学可能会为过去学生的学业成绩证书设立一个预言机。但是，存储证书的完整详细信息（细致到所修的课程和达到的成绩）是多余的。相反，证书的哈希就足够了。同样，政府可能希望将公民身份证放入以太坊平台，其中显然包含的细节需要保密。再次，散列数据（更仔细的做法是，在默尔克树中使用Salt）并且仅将根哈希存储在智能合约的存储中将是组织这种服务的有效方式。

下一个设置方式是*发布与订阅*，在这种预言机中，要对预期改变的数据（可能是定期和频繁地）提供有效的广播服务，预言机要么由链上的智能合约轮询，要么由链外守护进程监视和更新。此类别具有类似于RSS摘要或WebSub的模式，其中预言机使用新信息进行更新，并用标记表示新数据可供“订阅”的人使用。感兴趣的人必须将预言机轮询到检查最新信息是否已更改，或监听预言机合约的更新并在发生时采取行动。示例包括价格馈送、天气信息、经济或社会统计、交通数据等。在Web服务器领域，轮询效率非常低，但在区块链平台的对等环境中却不是这样：以太坊客户必须跟上所有状态更改，包括对合约存储的更改，因此轮询数据更改是对同步客户端的本地调用。以太坊事件日志使应用程序特别容易注意预言机更新，因此这种模式在某些方面甚至可以被视为“推送”服务。但是，如果轮询是通过智能合约完成的——这对于某些去中心化的应用可能是必要的（例如，在无法激活激励的情况下），则可能产生大量的gas支出。

“*请求/响应*”类别是最复杂的：这是数据空间太大而无法存储在智能合约中的情况，并且用户每次只需要整个数据集的一小部分。它也是数据提供商业务的适用模型。实际上，这样的预言机可以实现为链上智能合约系统，以及用于监视请求和检索、返回数据的链外基础结构。来自去中心化应用的数据请求通常是涉及许多步骤的异步过程。在这种模式中，首先，EOA与去中心化应用进行交互，从而与预言机智能合约中定义的功能进行交互。此函数启动对预言机的请求，除了可能包含回调函数和调度参数的补充信息之外，还使用相关参数详细说明所请求的数据。一旦验证了此事务，就可以将预言机请求视为预言机合约发出的EVM事件，或者作为状态更改；可以检索参数并用于执行链外数据源的实际查询。预言机可能还需要付款来处理请求，回调的gas支付以及访问所请求数据的权限。最后，结果数据由预言机所有者签名，证明在给定时间内的数据有效性，并在事务中传递给直接或通过预言机合约发出请求的去中心化应用。根据调度参数，预言机可以定期广播进一步更新数据的事务（例如，日终定价信息）。

请求与响应预言机的步骤可以总结如下：

1. 接收来自DApp的查询。
2. 解析查询。
3. 检查是否提供了付款和数据访问权限。
4. 从链外数据源检索相关数据（并在必要时加密）。
5. 使用包含的数据对事务进行签名。
6. 将事务广播到网络。
7. 安排任何进一步必要的交易，例如通知等。

一系列其他方案也是可能的，例如，可以从EOA请求数据并直接返回数据，从而无须使用预言机智能合约。

类似地，可以向支持物联网的硬件传感器发出请求和响应。因此，预言机可以是人、软件或硬件。此处描述的请求与响应模式常见于客户端与服务器体系结构。虽然这是一种有用的消息传递模式，允许应用程序进行双向对话，但在某些情况下这可能是不合适的。例如，在请求与响应模式下，需要预言机利率的智能债券可能必须每天请求数据，以确保利率始终是正确的。鉴于利率不经常变化，发布与订阅模式可能更合适这种情况，尤其是考虑到以太坊的有限带宽。

在发布与订阅模式中，发布者（在此上下文中是指预言机）不直接向接收者发送消息，而是将发布的消息分类到不同的类中。订阅者能够表达对一个或多个类的兴趣并仅检索那些感兴趣的消息。在这种模式下，预言机可能会在每次更改时将利率写入其自己的内部存储。多个订阅的DApp可以简单地从预言机合约中读取它，从而减少对网络带宽的影响，同时最大限度地降低存储成本。

在广播或多播模式中，预言机会将所有消息发布到信道，订阅合约将在各种订阅模式下收听信道。例如，预言机可能会将消息发布到加密货币汇率信道。订阅智能合约如果需要时间序列，例如移动平均计算，则可以请求信道的全部内容；另一个可能只需要现货价格计算最新利率。在预言机不需要知道订阅合约的身份的情况下，广播模式是合适的。

## 数据认证

即便我们假设被去中心化应用查询的数据源是权威的和值得信任的，仍然存在一个突出的问题：鉴于预言机以及“请求/响应”机制可能由多个实体来操作，我们如何才能信任这个机制呢？数据在传输过程中被篡改的可能性显然是存在的，所以，让链外方法可以证明返回数据的完整性是非常关键的。两种常见的数据认证方法是真实性证明（authenticity proof）以及可信执行环境（Trusted Execution Environment，TEE）。

真实性证明是用密码学证据证明数据没有被篡改过。基于许多证明技术（例如，数字签名证明），它们将需要的信任从数据传输者高效地转移到证明人（证明方法的提供者）。通过链上验证证据，智能合约可以在使用数据前验证数据的完整性。Oraclize（[http://www.oraclize.it/](http://www.oraclize.it/)）就是一个利用多种真实性证明的预言机服务的例子。现在以太坊主网查询数据时可以使用的其中一种证明方式是TLSNotary Proof。TLSNotary Proof允许客户端向第三方提交证据，证明客户端与某服务器之间发生了HTTPS网络流量。虽然HTTPS自身是安全的，但它并不支持数据签名。因此，TLSNotaryProof依赖于TLSNotary签名方案（通过PageSigner）。TLSNotaryProof利用了传输层安全协议（Transport Layer Security，TLS），这让TLS可以掌控密钥，在获取数据后给数据签名，并将数据分配给三方：服务器（预言机）、受审单位（Oraclize）以及审计方。Oraclize使用亚马逊网络服务器（AWS）虚拟机实例作为审计方，可以验证自实例化以来它没有被修改过。这一AWS实例存储着TLSNotary密文，密文让它可以提供诚实性证明。虽然这套方案在数据篡改上提供了比纯粹的“请求/响应”机制更高的安全保证，我们仍然需要假设亚马逊自己不会篡改虚拟机实例。

Town Crier（[http://www.towncrier.org/](http://www.towncrier.org/)）是一个基于可信执行环境的验证数据馈送预言机系统；这些方法采用基于硬件的安全区（security enclave）来验证数据完整性。Town Crier使用英特尔的SGX（Software Guard eXtensions）来保证对HTTPS查询的响应可以被验证为可信的。SGX也提供了完整性保证，使得在安全区中运行的应用受到CPU保护，不被其他进程篡改。它也提供了机密性，保证应用程序在安全区中运行时，其状态对其他进程来说是不可知的。最后，SGX通过生成应用程序确定在安全区中运行的数字签名（通过其构建结果的哈希值来安全地确定），让证明成为可能。通过验证这一数字签名，去中心化应用就可以证明Town Crier实例正在SGX安全区内安全地运行。这样就反过来证明了该实例没有被篡改过，由Town Crier发出的数据是可信的。此外，这种机密属性还让Town Crier可以处理隐私数据：使用Town Crier实例的公钥来加密数据查询请求。在安全区（如SGX）中运行预言机的“请求/响应”机制，我们完全可以认为Town Crier是在可信第三方硬件中安全地运行，保证了所请求的数据不受篡改地返回（假设我们相信Intel/SGX）。

## 计算性的预言机

迄今为止，我们只讨论请求和分发数据情境下的预言机。但是，预言机也可以用来执行任意计算。给定以太坊内在的区块gas上限和相对较贵的计算成本，是一个特别有用的功能。不只是传递数据请求的结果，计算预言机也可以用来执行带有一组输入的相关计算并返回计算结果，而这种计算可能无法在链上进行。举个例子，我们可以使用计算预言机来执行计算密集型的回归计算，以估计某个债券智能合约的收益。

如果你愿意信任集中而可审计的服务，你可以再次访问Oraclize。它们提供的服务允许去中心化应用请求在沙盒AWS虚拟机中执行的计算结果。AWS实例从包含在存档中的用户配置的Dockerfile创建可执行容器，该存档上载到星际文件系统（IPFS，参见第12章“数据存储”一节）。根据请求，Oraclize使用其哈希检索此存档，然后在AWS上初始化并执行Docker容器，传递将作为环境变量提供给应用程序的任何参数。容器化应用程序根据时间限制执行计算，并将结果写入标准输出，Oraclize可以对其进行检索并返回到去中心化应用。Oraclize目前在可审计的t2.microAWS实例上提供此服务，因此如果计算具有一些非常重要的值，则可以检查是否执行了正确的Docker容器。尽管如此，这不是一个真正去中心化的解决方案。

作为可验证预言机事实上的标准，“Cryptlet”的概念已经被规范化为微软更广泛的ESC框架的一部分。Cryptlet在一个密封安全区中运行，该密封安全区是从基础设施比如I/O中抽象出来的，并且附加上了CryptoDelegate，所以输入和输出消息都会自动签名、验证和证明。Cryptlet支持分布式交易，所以合约逻辑可以用具备ACID属性的方式来处理复杂的多步骤、多区块链交易以及外部系统交易。开发者因此可以创建用于智能合约中的可移植的、独立且具隐私性的事实解析。Cryptlet遵循下列格式：

```Solidity
public class SampleContractCryptlet : Cryptlet
    {
        public SampleContractCryptlet(Guid id, Guid bindingId, string name,
            string address, IContainerServices hostContainer, bool contract)
            : base(id, bindingId, name, address, hostContainer, contract)
        {
            MessageApi = new CryptletMessageApi(GetType().FullName,
                new SampleContractConstructor())
```

TrueBit（[https://truebit.io](https://truebit.io/)）是一个可扩展和可验证的链外计算解决方案。它引入了一个解决者和验证者系统，它们被激励各自执行计算和验证这些计算。如果一个计算结果受到挑战，链上就会相应执行对该计算子集的迭代验证进程——这是一种类型的“验证游戏”。验证游戏会进行几轮，每一轮都会递归地查验相关计算的更小子集。挑战充分细分之后，游戏的终局便到来，法官（以太坊矿工）便可在链上最终裁定相关挑战是否合理。实际上，TrueBit是计算市场的一种实现，去中心化应用因此可以为可验证计算支付；计算虽然是在链外执行的，但依靠以太坊来强制执行验证者游戏的规则。理论上来说，这让免信任型智能合约安全地执行任意计算任务。

像TrueBit这样的系统有很多应用，从机器学习到任意工作量证明的验证。后者的其中一个例子是Doge Ethereum桥接，它利用TrueBit来验证狗狗币（Dogecoin）的工作量证明算法Scrypt，这是一种强内存需求且计算密集的函数，它不可能在以太坊区块gas上限内计算完成。通过在TrueBit执行这种验证，在以太坊Rinkeby测试网络上用智能合约安全地验证狗狗币交易便成为可能。

## 去中心化预言机

上面列举出的所有机制描述的都是中心化的预言机系统，都需要依赖可信的权威。虽然它们可以为许多应用服务，它们的存在仍然意味着以太坊网络中的单点故障。围绕着去中心化预言机，人们已经提出了很多计划：去中心化预言机可以用于保证数据可得性，还可搭配链上数据汇总系统创建独立数据提供者网络。

Chainlink（[https://chain.link](http://chain.link)）已经提出了一种去中心化预言机网络，由三个关键智能合约（声誉合约、订单匹配合约、数据汇总合约）以及数据提供者的链外注册表组成。声誉合约用来跟踪数据提供者的表现。声誉合约中的分数会更新到链外注册表中。订单匹配合约会从使用声誉合约的预言机中选择竞标者，并最终确定服务层级要约（[Service Level Agreement](https://github.com/smartcontractkit/chainlink/wiki/Protocol-Information)，SLA），其中包含了查询参数和要求的预言机数量。这也意味着数据购买者不会直接与个体预言机交易。数据汇总合约会从多个预言机处收集响应（使用“commit reveal”模式提交），计算查询的最终总结果，然后将结果反馈回声誉合约。

这样的去中心化方案要面临的其中一个重大挑战是：构建数据汇总函数。Chainlink提议计算响应附加权重，这样就可以为每一个预言机响应记录有效性分数。此处，发现一个“无效”的分数并不是毫无价值的，因为它建立在：（以对统计提供的响应的偏离来度量）过于偏远的数据点是不正确的这一前提之上。基于某一预言机响应围绕响应分布的位置来计算有效性分数有一定风险，具体表现为惩罚偏离平均值的正确答案。因此，Chainlink提供汇总函数的标准集合，但也支持使用定制化的汇总合约。

一个相关的想法是谢林币协议。其中，多个参与者记录数值，这些数值的中位数会被当成“正确”答案。记录者必须先质押保证金，这些保证金会根据它与中位数的接近程度重新分配，由此可以激励人们记录与其他人提供的值相近的值。这样共同的数值，也就是所谓的“谢林点”，预计会接近真实值，因为真实数值是响应者协作中所围绕的自然而明显的目标。

Jason Teutsch最近提出一种新型的去中心化链外数据可得性预言机。这种设计利用了一条专用的工作量证明区块链，后者可以在给定时间内正确地记录登记过的数据是否可得。矿工会尝试下载、存储和传播所有新近登记的数据，因此保证数据在本地是可得的。这样的系统是昂贵的，因为每一个挖矿节点都要存储和传播所有登记过的数据。该系统可以通过在登记期结束后释放数据来重复利用存储空间。

## Solidity中的预言机客户端接口

代码11-1是一个Solidity示例，展示了Oraclize如何从API不断获取ETH/USD价格并以可用的方式存储结果。

代码11-1：使用Oraclize从外部来源更新ETH/USD汇率

```Solidity
/*
    ETH/USD price ticker leveraging CryptoCompare API

    This contract keeps in storage an updated ETH/USD price,
    which is updated every 10 minutes.
    */

pragma solidity ^0.4.1;
import "github.com/oraclize/ethereum-api/oraclizeAPI.sol";

/*
    "oraclize_" prepended methods indicate inheritance from "usingOraclize"
    */
contract EthUsdPriceTicker is usingOraclize {

    uint public ethUsd;

    event newOraclizeQuery(string description);
    event newCallbackResult(string result);

    function EthUsdPriceTicker() payable {
        // signals TLSN proof generation and storage on IPFS
        oraclize_setProof(proofType_TLSNotary | proofStorage_IPFS);

        // requests query
        queryTicker();
    }

    function __callback(bytes32 _queryId, string _result, bytes _proof) public {
        if (msg.sender != oraclize_cbAddress()) throw;
        newCallbackResult(_result);

        /*
            * Parse the result string into an unsigned integer for on-chain use.
            * Uses inherited "parseInt" helper from "usingOraclize", allowing for
            * a string result such as "123.45" to be converted to uint 12345.
            */
        ethUsd = parseInt(_result, 2);

        // called from callback since we're polling the price
        queryTicker();
    }

    function queryTicker() public payable {
        if (oraclize_getPrice("URL") > this.balance) {
            newOraclizeQuery("Oraclize query was NOT sent, please add some ETH
                to cover for the query fee");
        } else {
            newOraclizeQuery("Oraclize query was sent, standing by for the
                answer...");

            // query params are (delay in seconds, datasource type,
            // datasource argument)
            // specifies JSONPath, to fetch specific portion of JSON API result
            oraclize_query(60 * 10, "URL",
                "json(https://min-api.cryptocompare.com/data/price?\
                fsym=ETH&tsyms=USD,EUR,GBP).USD");
        }
    }
}
```

为结合Oraclize，EthUsdPriceTicker合约必须是usingOraclize合约的子合约；后者是在oraclizeAPI文件中定义好的。数据请求会由usingOraclize合约内置的oraclize_query函数发起。这是一个重载函数，预计至少需要两个参数：

- 支持使用的数据源，如URL、WolframAlpha、IPFS或计算。
- 为给定数据源设定的参数，可能包括JSON或XML解析助手的使用。

数据查询的价格会由queryTicker函数执行。为执行数据查询请求，Oraclize要求用支付一小笔费用，用于补偿传输和处理结果到_callback函数过程中发生的gas费用以及为服务支付的额外费用。费用的数额视数据源和要求的可信证明类型（如果有所指定的话）而定。一旦检索了数据，_callback函数就会由Oraclize控制的许可账户调用；这一过程会传入响应值和唯一的queryId参数，后者可以用于处理和跟踪来自Oraclize的多个待定的回调。

金融数据提供者ThomsonReuters也为以太坊提供了一项名为“BlockOneIQ”的预言机服务，让运行在私有或许可网络上的智能合约可以请求市场和参考数据。代码112是该预言机的交互接口，以及用于发起请求的客户端智能合约。

代码11-2：合约调用BlockOneIQ服务以获取市场数据

```Solidity
pragma solidity ^0.4.11;

contract Oracle {
    uint256 public divisor;
    function initRequest(
        uint256 queryType, function(uint256) external onSuccess,
        function(uint256
    ) external onFailure) public returns (uint256 id);
    function addArgumentToRequestUint(uint256 id, bytes32 name, uint256 arg) public;
    function addArgumentToRequestString(uint256 id, bytes32 name, bytes32 arg)
        public;
    function executeRequest(uint256 id) public;
    function getResponseUint(uint256 id, bytes32 name) public constant
        returns(uint256);
    function getResponseString(uint256 id, bytes32 name) public constant
        returns(bytes32);
    function getResponseError(uint256 id) public constant returns(bytes32);
    function deleteResponse(uint256 id) public constant;
}

contract OracleB1IQClient {

    Oracle private oracle;
    event LogError(bytes32 description);

    function OracleB1IQClient(address addr) public payable {
        oracle = Oracle(addr);
        getIntraday("IBM", now);
    }

    function getIntraday(bytes32 ric, uint256 timestamp) public {
        uint256 id = oracle.initRequest(0, this.handleSuccess, this.handleFailure);
        oracle.addArgumentToRequestString(id, "symbol", ric);
        oracle.addArgumentToRequestUint(id, "timestamp", timestamp);
        oracle.executeRequest(id);
    }

    function handleSuccess(uint256 id) public {
        assert(msg.sender == address(oracle));
        bytes32 ric = oracle.getResponseString(id, "symbol");
        uint256 open = oracle.getResponseUint(id, "open");
        uint256 high = oracle.getResponseUint(id, "high");
        uint256 low = oracle.getResponseUint(id, "low");
        uint256 close = oracle.getResponseUint(id, "close");
        uint256 bid = oracle.getResponseUint(id, "bid");
        uint256 ask = oracle.getResponseUint(id, "ask");
        uint256 timestamp = oracle.getResponseUint(id, "timestamp");
        oracle.deleteResponse(id);
        // Do something with the price data
    }

    function handleFailure(uint256 id) public {
        assert(msg.sender == address(oracle));
        bytes32 error = oracle.getResponseError(id);
        oracle.deleteResponse(id);
        emit LogError(error);
    }

}
```

使用initRequest函数启动数据请求，该函数允许指定查询类型（在此示例中是对日内价格的请求）以及两个回调函数。这将返回一个uint256标识符，然后可用于提供其他参数。addArgumentToRequestString函数用于指定路透代码表（RIC），此处为IBM库存，addArgumentToRequestUint允许指定时间戳。现在，传入block.timestamp的别名将检索IBM的当前价格。然后由executeRequest函数执行该请求。处理完请求后，预言机合约将使用查询标识符调用onSuccess回调函数，从而允许检索结果数据；如果检索失败，onFailure回调函数将返回错误代码。成功检索的可用字段包括open（开盘价）、high（最高价）、low（最低价）、close（收盘价）和bid/ask（买/卖价）。

## 总结

正如你所看到的，预言机为智能合约提供了至关重要的服务：它们将外部事实带入合约执行。当然，预言机也会带来很大的风险：如果它们是受信任的来源并且可能受到损害，可能导致它们提供的智能合约的执行受损。

一般来说，在考虑使用预言机时要非常小心信任模型。如果你认为预言机可以信任，那么你可能会通过将其暴露给潜在的错误输入来破坏智能合约的安全性。也就是说，如果仔细考虑安全假设，那么预言机会非常有用。

去中心化的预言机可以解决其中一些问题，并为以太坊智能合约提供无信任的外部数据。谨慎选择，你就可以开始探索以太坊与预言机提供的“真实世界”之间的桥梁。

（希）安德烈亚斯·M.安东波罗斯. 精通以太坊：开发智能合约和去中心化应用 (OReilly精品图书系列) (Chinese Edition) (Kindle Locations 5007-5013). Kindle Edition.