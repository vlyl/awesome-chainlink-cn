# Chainlink基本原理

本文我们来从技术上简述一下Chainlink的基本原理。如果用最短的一句话解释什么是Chainlink，可以说Chainlink一个**去中心化**的**预言机**项目，所以为了理解Chainlink的工作原理，我们首先要明白什么是预言机。

# 预言机

预言机的英文为Oracle，和著名的数据库服务提供商Oracle（甲骨文）重名，但是两者除了名字相同以为并没有任何关系。Oracle这个单词是什么意思，下面是我在[vocabulary.com](https://www.vocabulary.com/dictionary/oracle)上查到的Oracle的含义：

> Back in ancient times, an oracle was someone who offered advice or a prophecy thought to have come directly from a divine source. In modern usage, any good source of information can be called an oracle.

中文的大概意思是：在古代，oracle是一个提出建议或预言的人，他的建议或预言被认为是直接来自于神。在现代用法中，**任何好的信息来源都可以称为oracle**。

这样就不难理解了，Oracle传达了万能全知的神的旨意，而甲骨文最初就是用来占卜吉凶时的记录，也在当时也被认为是神谕，传达了神的意思。说以不管是“预言机”还是“甲骨文”都表达了“信息源”的意思。

计算机领域内的预言机一词，最早是图灵提出的。图灵在图灵机（Turing Machine）的基础上，加入了一个称为预言者（oracle）的黑盒，组成了预言机（Oracle Machine）。所谓预言者，是一个可以回答特定问题集合的实体。即它可以向图灵机系统内部输入信息，帮助图灵机完成运算。以太坊的智能合约是“图灵完备（Turing Complete）”的，某种意义上可以看做一个图灵机，所以以太坊的设计者借鉴这个概念，把向“图灵完备的智能合约”这个图灵机输入信息的也被称为预言机oracle。所以说“预言机”这个名字并不是区块链技术领域内的独创概念，它来源于非常早期的计算机抽象设计，在密码学等领域内也都有类似的概念。

而在区块链领域，预言机被认为是可以为智能合约提供外部数据源的系统。从传统技术架构方面来看，预言机是连接智能合约与区块链外部世界的**中间件（middleware）**，是区块链重要的基础设施，它的作用是为区块链上的智能合约（Smart Contract）提供数据信息的。

正如以太坊的定义，区块链是一个交易驱动的状态机（a transaction-based state machine），它能做的事情非常简单，就是通过向区块链提交事务/交易（transaction），来将区块链从一个状态转变成另一个状态。为了保持共识，EVM的执行过程必须完全确定，并且仅基于以太坊状态和签名交易的共享上下文。这产生了两个特别重要的后果：一个是EVM和智能合约没有内在的随机性来源；另一个是外部数据只能作为交易的数据载荷引入。用通俗的话讲，区块链没有主动获取数据的能力，它能用的只有区块链自己本身的数据。数据的缺失导致智能合约的应用范围非常少，目前大部分的应用都是围绕着token来展开的。

区块链的确定性的意思是，在任何节点上，只要连入到区块链的分布式网络中，它就可以同步所有的历史区块，回放出一套完全相同的账本。换句话说：在没有互联网连接的情况下，给定完整的块，节点必须能够从头开始重新创建区块链的最终状态。如果账本在形成过程中，依赖于某个外部的API调用结果，那在不同时间不同环境下回放的结果就会不一样。这种情况是区块链所不允许的，所以区块链在设计之初就没有网络调用。

那么要实现向区块链提供数据，应该怎么做呢？区块链能留下的只有账本，而区块链所能输入的只有交易。我们就从这两个方面入手。

![Chainlink/Untitled.png](../img/Chainlink-fundamental1.png)

几乎每一个合约系统，都会有事件记录的功能，比如以太坊中的EventLog功能。

下面我们通过一个例子，来介绍一下预言机的基本原理。我们在以太坊链上建立一个用户合约，它需要获取到某个城市的气温数据。当然，智能合约自己是无法获取到这个发生于链下真实世界中的数据信息的，需要借助预言机来实现。智能合约将需要获取天气温度的的城市写入到EventLog中，链下我们会启动一个进程，监听并订阅这个事件日志，获取到智能合约的请求之后，将指定城市的温度，通过提交transaction的方式，调用合约中的回填方法，提交到智能合约中。

声明：以下代码仅供演示预言机原理，没有做参数检测和错误处理，请不要在生产环境中使用。

消费者合约：

```solidity
    contract WeatherOracle {
    		// 用户存储预言机提交的天气数值
        uint256 public temperature;
    
    		// 定义事件
        event RequestTemperature (bytes city);
    
    		// 发出获取请求，即发出一个事件日志
        function requestTemperature (string memory _city) public {
            emit RequestTemperature(bytes(_city));
        }
    
    		// 预言机回调方法，预言机获取到数据后通过这个方法将数据提交到链上
        function updateWeather (uint256 _temperature) public {
            temperature = _temperature;
        }
    }
```

上面的代码非常简单，定义了一个变量用来存储结果，一个方法用于发出请求，一个方法用于接收结果。

链下，我们启动一个进程，以订阅topic的方式获取日志信息，之后通过构建一个transaction，提交一个结果到合约中。

```solidity
    func SubscribeEventLog() {
    	topic := crypto.Keccak256([]byte("RequestTemperature(bytes)"))
    	query := ethereum.FilterQuery{
    		Topics: [][]common.Hash{
    			{
    				common.BytesToHash(topic),
    			},
    		},
    	}
    
    	// 订阅相关主题的日志事件
    	events := make(chan types.Log)
    	sub, err := EthClient.SubscribeFilterLogs(ctx, query, events)
    
    	// 加载合约的ABI文件
    	ta, err := abi.JSON(strings.NewReader(AbiJsonStr))
    
      // 监听事件订阅
    	for {
    		select {
    		case err := <-sub.Err():
    			log.Error(err)
    			break
    		case ev := <-events:
    			// 获取到订阅的消息
    			ej, _ := ev.MarshalJSON()
    			log.Info(string(ej))
    
    			// 解析数据
    			var sampleEvent struct {
    				City []byte
    			}
    			err = ta.Unpack(&sampleEvent, "RequestTemperature", ev.Data)
    			log.Info(string(sampleEvent.City))
    
    			// 构建交易提交结果，需要提供私钥用于签署交易
    			CallContract("b7b502b...164b42c")
    		}
    	}
    }
```

```solidity
    func CallContract(keyStr string) {
    	addr := PrivateKeyToAddress(keyStr)
    	nonce, err := EthClient.PendingNonceAt(ctx, addr)
    
    	gasPrice, err := EthClient.SuggestGasPrice(ctx)
    
    	privateKey, err := crypto.HexToECDSA(keyStr)
    
    	auth := bind.NewKeyedTransactor(privateKey)
    	auth.Nonce = big.NewInt(int64(nonce))
    	auth.Value = big.NewInt(0)
    	auth.GasLimit = uint64(300000)
    	auth.GasPrice = gasPrice
    
    	instance, err := event.NewEvent(common.HexToAddress("0x8A421906e9562AA1c71e5a32De1cf75161C5A463"), EthClient)
    
    	// 调用合约中的updateWeather方法，回填数据"29"
    	tx, err := instance.UpdateWeather(auth, big.NewInt(29))
    
    	log.Info(tx.Hash().Hex())
    }
```

用一个图来展示这个过程：

![Chainlink/Untitled%201.png](../img/Chainlink-fundamental2.png)

# Chainlink

Chainlink是一个去中心化的预言机项目，它的作用就是以最安全的方式向区块链提供现实世界中产生的数据。Chainlink在基本的预言机原理的实现方式之上，围绕LINK token通过经济激励建立了一个良性循环的生态系统。Chainlink预言机需要通过LINK token的转账来实现触发。

LINK是以太坊网络上的ERC677合约，关于各类ERC token的区别，请参考这篇[文章](http://blockchainers.org/index.php/2018/02/08/token-erc-comparison-for-fungible-tokens/)。

在《精通以太坊（Matering Ethereum）》一书中，提出了三种预言机的设计模式，分别是

- 立即读取（immediate-read）
- 发布/订阅（publish–subscribe）
- 请求/响应（request–response）

而基于LINK ERC677 token完成的预言机功能，就属于其中的**请求/响应**模式。这是一种较为复杂的模式，上图中展示的是一个不含有聚合过程的简单请求/相应流程。

![Chainlink/Untitled%202.png](../img/Chainlink-fundamental3.png)

我们以Chainlink提供的`TestnetConsumer`合约中的一个`requestEthereumPrice` 方法为例来简单讲一下请求响应的流程。这个函数定义如下：

```solidity
    function requestEthereumPrice(address _oracle, string _jobId)
        public
        onlyOwner
      {
        Chainlink.Request memory req = buildChainlinkRequest(stringToBytes32(_jobId), this, this.fulfillEthereumPrice.selector);
        req.add("get", "https://min-api.cryptocompare.com/data/price?fsym=ETH&tsyms=USD");
        req.add("path", "USD");
        req.addInt("times", 100);
        sendChainlinkRequestTo(_oracle, req, ORACLE_PAYMENT);
      }
```

它所实现的功能就是从指定的API(cryptocompare)获取ETH/USD的交易价格。函数传入的参数是指定的oracle地址和jobId。将一些列的请求参数组好后，调用`sendChainlinkRequestTo` 方法将请求发出。`sendChainlinkRequestTo`是定义在Chainlink提供的[库](https://github.com/smartcontractkit/chainlink/evm/contracts/ChainlinkClient.sol)中的一个接口方法，定义如下：

```solidity
     /**
       * @notice 向指定的oracle地址创建一个请求
       * @dev 创建并存储一个请求ID, 增加本地的nonce值, 并使用`transferAndCall` 方法发送LINK，
       * 创建到目标oracle合约地址的请求
       * 发出 ChainlinkRequested 事件.
       * @param _oracle 发送请求至的oracle地址
       * @param _req 完成初始化的Chainlink请求
       * @param _payment 请求发送的LINK数量
       * @return 请求 ID
       */
      function sendChainlinkRequestTo(address _oracle, Chainlink.Request memory _req, uint256 _payment)
        internal
        returns (bytes32 requestId)
      {
        requestId = keccak256(abi.encodePacked(this, requests));
        _req.nonce = requests;
        pendingRequests[requestId] = _oracle;
        emit ChainlinkRequested(requestId);
        require(link.transferAndCall(_oracle, _payment, encodeRequest(_req)), "unable to transferAndCall to oracle");
        requests += 1;
    
        return requestId;
      }
```

其中`link.transferAndCall`方法即是ERC677定义的token转账方法，与ERC20的`transfer`方法相比，它多了一个data字段，可以在转账的同时携带数据。这里就将之前打包好的请求数据放在了data字段，跟随转账一起发送到了oracle合约。`transferAndCall` 方法定义如下：

```solidity
    /**
      * @dev 将token和额外数据一起转移给一个合约地址
      * @param _to 转移到的目的地址
      * @param _value 转移数量
      * @param _data 传递给接收合约的额外数据
      */
      function transferAndCall(address _to, uint _value, bytes _data)
        public
        returns (bool success)
      {
        super.transfer(_to, _value);
        Transfer(msg.sender, _to, _value, _data);
        if (isContract(_to)) {
          contractFallback(_to, _value, _data);
        }
        return true;
      }
```

其中的`Transfer(msg.sender, _to, _value, _data);`是发出一个事件日志：

```solidity
    event Transfer(address indexed from, address indexed to, uint value, bytes data);
```

将这次转账的详细信息（发送方、接收方、金额、数据）记录到日志中。

Oracle合约在收到转账之后，会触发`onTokenTransfer`方法，该方法会检查转账的有效性，并通过发出`OracleRequest`事件记录更为详细的数据信息：

```solidity
    event OracleRequest(
        bytes32 indexed specId,
        address requester,
        bytes32 requestId,
        uint256 payment,
        address callbackAddr,
        bytes4 callbackFunctionId,
        uint256 cancelExpiration,
        uint256 dataVersion,
        bytes data
      );
```

这个日志会在oracle合约的日志中找到，如图中下方所示。链下的节点会订阅该主题的日志，在获取到记录的日志信息之后，节点会解析出请求的具体信息，通过网络的API调用，获取到请求的结果。之后通过提交事务的方式，调用Oracle合约中的`fulfillOracleRequest`方法，将数据提交到链上。`fulfillOracleRequest`定义如下：

```solidity
     /**
       * @notice 由Chainlink节点调用来完成请求
       * @dev 提交的参数必须是`oracleRequest`方法所记录的哈希参数
    	 * 将会调用回调地址的回调函数，`require`检查时不会报错，以便节点可以获得报酬
       * @param _requestId 请求ID必须与请求者所匹配
       * @param _payment 为Oracle发放付款金额 (以wei为单位)
       * @param _callbackAddress 完成方法的回调地址
       * @param _callbackFunctionId 完成方法的回调函数
       * @param _expiration 请求者可以取消之前节点应响应的到期时间
       * @param _data 返回给消费者合约的数据
       * @return 外部调用成功的状态值
       */
      function fulfillOracleRequest(
        bytes32 _requestId,
        uint256 _payment,
        address _callbackAddress,
        bytes4 _callbackFunctionId,
        uint256 _expiration,
        bytes32 _data
      )
        external
        onlyAuthorizedNode
        isValidRequest(_requestId)
        returns (bool)
      {
        bytes32 paramsHash = keccak256(
          abi.encodePacked(
            _payment,
            _callbackAddress,
            _callbackFunctionId,
            _expiration
          )
        );
        require(commitments[_requestId] == paramsHash, "Params do not match request ID");
        withdrawableTokens = withdrawableTokens.add(_payment);
        delete commitments[_requestId];
        require(gasleft() >= MINIMUM_CONSUMER_GAS_LIMIT, "Must provide consumer enough gas");
        // All updates to the oracle's fulfillment should come before calling the
        // callback(addr+functionId) as it is untrusted.
        // See: https://solidity.readthedocs.io/en/develop/security-considerations.html#use-the-checks-effects-interactions-pattern
        return _callbackAddress.call(_callbackFunctionId, _requestId, _data); // solhint-disable-line avoid-low-level-calls
      }
```

这个方法会在进行一系列的检验之后，会将结果通过之前记录的回调地址与回调函数，返回给消费者合约：

```solidity
    _callbackAddress.call(_callbackFunctionId, _requestId, _data);
```

这样一次请求就全部完成了。

# 总结

本文从预言机的概念开始，通过一个简单的获取ETH价格的例子，讲解了请求/响应模式的Chainlink预言机的基本过程，希望对你理解预言机与Chainlink的运行原理有所帮助。


参考:
https://medium.com/@liyunlong518/%E5%88%9B%E5%BB%BA%E4%BD%A0%E7%9A%84%E7%AC%AC%E4%B8%80%E4%B8%AA%E4%BB%A5%E5%A4%AA%E5%9D%8A%E9%A2%84%E8%A8%80%E6%9C%BA-df53e50cc2d5