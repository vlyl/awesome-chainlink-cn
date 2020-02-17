# 为什么LINK使用ERC-677标准发行token

等等，LINK不是ERC20吗，怎么又成了ERC677了？

别急，我们先从ERC20开始说起。

ERC20是一套协议标准，代码角度来说就是一套接口API。在这个协议标准下，只要实现了协议标准所规定的方法，都可以作为ERC20代币的实现。协议规定必须实现的方法有：

```
// 1. 代币发行总量
function totalSupply() public view returns (uint256)

// 2. _owner账户的代币余额
function balanceOf(address _owner) public view returns (uint256 balance)

// 3. 转移_value数量的代币到_to地址
function transfer(address _to, uint256 _value) public returns (bool success)

// 4. 从_address地址转移_value数量的代币到_to地址
function transferFrom(address _from, address _to, uint256 _value) public returns (bool success)

// 5. 允许_spender可以提取总额为_value数量的代币，提取不限次数
function approve(address _spender, uint256 _value) public returns (bool success)

// 6. 返回_spender还可以从_owner提取的代币数量
function allowance(address _owner, address _spender) public view returns (uint256 remaining)
```

除了这几个方法，ERC20还规定了两个事件：

```
// 当成功转移token时，触发Transfer事件，记录转账的发送方、接收方和转账金额
event Transfer(address indexed _from, address indexed _to, uint256 _value)

// 当调用approval函数成功时，触发Approval事件，记录批准的所有方、获取方和批准金额
event Approval(address indexed _owner, address indexed _spender, uint256 _value)
```

以上基本上就是ERC20代币的全部内容了。由于LINK代币不仅仅是代币，还承担了链上与链下数据传递的功能，所以如果使用ERC20代币的标准无法满足这个需求，于是LINK选择了ERC677协议标准来实现。

ERC677标准是ERC20的一个扩展，它继承了ERC20的所有方法和事件，由Chainlink的CTO Steve Ellis首次[提出](https://github.com/ethereum/EIPs/issues/677)。ERC677除了包含了ERC20的所有方法和事件之外，增加了一个`transferAndCall` 方法：

```
function transferAndCall(address receiver, uint amount, bytes data) returns (bool success)
```

可以看到，这个方法比ERC20中的`transfer`方法多了一个`data`字段，这个字段用于在转账的同时，携带用户自定义的数据。在方法调用的时候，还会触发`Transfer(address,address,uint256,bytes)` 事件，记录下发送方、接收方、转账金额以及附带数据。完成转账和记录日志之后，代币合约还会调用接收合约的`onTokenTransfer`方法，用来触发接收合约的逻辑。这就要就接收ERC677代币的合约必须实现`onTokenTransfer`方法，用来给代币合约调用。onTokenTransfer方法定义如下：
```
function onTokenTransfer(address from, uint256 amount, bytes data) returns (bool success)
```

接收合约就可以在这个方法中定义自己的业务逻辑，可以在发生转账的时候自动触发。换句话说，智能合约中的业务逻辑，可以通过代币转账的方式来触发自动运行。这就给了智能合约的应用场景有了很大的想象空间。比如LINK的token[合约](https://etherscan.io/address/0x514910771af9ca656af840dff83e8264ecf986ca#code)就是一个ERC677合约，而Chainlink的Oracle合约，是一个可以接收ERC677的[合约](https://etherscan.io/address/0x64fe692be4b42f4ac9d4617ab824e088350c11c2#code)，它含有onTokenTransfer方法，可以在收到LINK的转账的时候执行预言机相关的业务逻辑。

LINK token contract:

```
...

  /**
  * @dev 转移token到合约地址，并携带额外数据
  * @param _to 转到的地址
  * @param _value 转账金额
  * @param _data 传递给接受合约的额外数据
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

...
```

Oracle 合约：

```
...

  /**
    * @notice 在LINK通过`transferAndCall`方法发送到合约时被调用
    * @dev 负载数据的前两个字节会被`_sender`和 `_amount`的值覆盖来保证正确性。并会调用oracleRequest方法
    * @param _sender 发送方地址
    * @param _amount 发送的LINK数量(单位是wei)
    * @param _data 交易的负载数据
    */
  function onTokenTransfer(
    address _sender,
    uint256 _amount,
    bytes _data
  )
    public
    onlyLINK
    validRequestLength(_data)
    permittedFunctionsForLINK(_data)
  {
    assembly {
      // solium-disable-next-line security/no-low-level-calls
      mstore(add(_data, 36), _sender) // ensure correct sender is passed
      // solium-disable-next-line security/no-low-level-calls
      mstore(add(_data, 68), _amount)    // ensure correct amount is passed
    }
    // solium-disable-next-line security/no-low-level-calls
    require(address(this).delegatecall(_data), "Unable to create request"); // calls oracleRequest
  }

...

```

总结：

LINK代币合约是ERC677合约，它是ERC20合约的一个扩展，兼容ERC20协议标准。它可以在转账时携带数据，并触发接收合约的业务逻辑，这一特点可以帮助智能合约扩大应用场景。

参考：

[https://eips.ethereum.org/EIPS/eip-20](https://eips.ethereum.org/EIPS/eip-20)

[http://blockchainers.org/index.php/2018/02/08/token-erc-comparison-for-fungible-tokens/](http://blockchainers.org/index.php/2018/02/08/token-erc-comparison-for-fungible-tokens/)

[https://github.com/ethereum/EIPs/issues/677](https://github.com/ethereum/EIPs/issues/677)

[https://etherscan.io/address/0x514910771af9ca656af840dff83e8264ecf986ca#code](https://etherscan.io/address/0x514910771af9ca656af840dff83e8264ecf986ca#code)

[https://etherscan.io/address/0x64fe692be4b42f4ac9d4617ab824e088350c11c2#code](https://etherscan.io/address/0x64fe692be4b42f4ac9d4617ab824e088350c11c2#code)

作者：Shannon@Chainlink（李云龙）
