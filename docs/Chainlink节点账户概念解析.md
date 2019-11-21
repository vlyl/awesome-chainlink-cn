# Chainlink节点账户概念解析

在部署运行Chainlink节点预言机合约的过程中，会涉及到很多的地址与账户，今天我们就来解释一下这些地址和账户分别的做什么用的。

# 合约相关

首先介绍一下以太坊账户相关的基础知识。

我们知道，以太坊上的账户分为外部账户（EOA, externally owned accounts）和合约账户（CA, contract accounts），外部账户就是我们普通用户掌握私钥的账户，可以用来存储、转账代币，也可以用来创建部署智能合约。在创建智能合约之后，合约也会拥有一个地址，这个地址和外部账户的地址在形式上没有区别（都是0x开头的16进制字符串），但是合约账户没有私钥，它通过外部账户提交transaction的方式去调用。虽然合约账户没有私钥，但是合约账户却可以持有资金，它的资金和一些重要的权限操作可以被其拥有者owner所控制。一般来说，owner是创建合约的外部账户，但是owner也可以被最初的创建者转移给其他账户。

我们在部署节点的时候，都会部署一个代表我们节点在链上执行request和fulfill的oracle合约。我们**部署合约所用到的外部账户，其私钥必须妥善保管**，因为它对于oracle合约有强有力的控制权，可以设置节点调用的权限，更重要的是它可以控制合约账户所持有的资金，具体来说就是LINK token。

我们可以看这个oracle合约中

      /**
       * @notice Sets the fulfillment permission for a given node. Use `true` to allow, `false` to disallow.
       * @param _node The address of the Chainlink node
       * @param _allowed Bool value to determine if the node can fulfill requests
       */
      function setFulfillmentPermission(address _node, bool _allowed) external onlyOwner {
        authorizedNodes[_node] = _allowed;
      }
    
      /**
       * @notice Allows the node operator to withdraw earned LINK to a given address
       * @dev The owner of the contract can be another wallet and does not have to be a Chainlink node
       * @param _recipient The address to send the LINK token to
       * @param _amount The amount to send (specified in wei)
       */
      function withdraw(address _recipient, uint256 _amount)
        external
        onlyOwner
        hasAvailableFunds(_amount)
      {
        withdrawableTokens = withdrawableTokens.sub(_amount);
        assert(LinkToken.transfer(_recipient, _amount));
      }
    
      /**
       * @notice Displays the amount of LINK that is available for the node operator to withdraw
       * @dev We use `ONE_FOR_CONSISTENT_GAS_COST` in place of 0 in storage
       * @return The amount of withdrawable LINK on the contract
       */
      function withdrawable() external view onlyOwner returns (uint256) {
        return withdrawableTokens.sub(ONE_FOR_CONSISTENT_GAS_COST);
      }

source: [https://github.com/smartcontractkit/chainlink/blob/develop/evm/contracts/Oracle.sol#L187](https://github.com/smartcontractkit/chainlink/blob/develop/evm/contracts/Oracle.sol#L187)

setFulfillmentPermission、withdraw、withdrawable三个方法都是只有owner（所有者）才能调用的。其中withdraw方法，可以理解为一个提币的方法，它将**合约账户持有的LINK token转移到其他账户**。所以**owner账户非常重要，一定要妥善保管**。

在提币的时候，可以先调用`withdrawable`方法，查看可提取的LINK token数量。如果想全部提取出来的话，就复制`withdrawable` 返回的数值到`withdrable`方法的参数中。需要注意的是，调用这两个方法的账户必须是这个合约的所有者。

# 节点相关

我们在按照文档[https://docs.chain.link/docs/running-a-chainlink-node](https://docs.chain.link/docs/running-a-chainlink-node)部署节点的时候，也会遇到很多账户。我们以docker方式启动为例，介绍一下这些账户的作用。

节点拥有一个以太坊的外部账户，这个账户会持有一定数量的ETH，用于提交调用oracle合约的事务时支付以太坊网络的交易费用。这个账户是在初次启动chainlink的实例的时候生成的。

我们在第一次启动chainlink示例的时候，比如在执行

`cd ~/.chainlink-ropsten && docker run -p 6688:6688 -v ~/.chainlink-ropsten:/chainlink -it --env-file=.env smartcontract/chainlink local n`

命令之后，首先会要求我们输入两遍密码（输入和确认），这个密码其实就是节点所拥有的以太账户的keystore的passphrase，必须要牢记，如果丢失了无法找回。

如果我们想把节点的账户地址上的ETH提出来应该怎么操作呢。我们需要找到这个keystore。如果你是按照官方文档的方式创建的节点，keystore会保存在你的节点所在的服务器的~/.chainlink/tempkey目录下（即Chainlink节点运行的主目录下的tempkey目录，请跟据自己节点的部署情况更改路径）。需要注意想要查看keystore内容需要你有服务器的sudo权限。拿到keystore后，就可以在你喜欢的以太坊钱包上用上面提到的密码导入了。

第一次启动chainlink示例的时候，在输入keystore的密钥之后，还会要求你输入一个账户和密码，这个账户密码是chainlink提供的web管理界面的登录用户名密码。这个账户的密码可以按照这篇文档提供的方式来修改。[https://docs.chain.link/docs/miscellaneous#section-change-your-api-password](https://docs.chain.link/docs/miscellaneous#section-change-your-api-password)

# 总结

本文我们介绍了Chainlink节点运营相关的4个账户，分别是

- **部署合约的外部账户：**用于部署Oracle合约，默认情况下是Oracle合约的Owner（所有者），掌管着账户本身的资金和Oracle合约所持有的资金
- **Oracle合约账户：**没有私钥，其持有的资金，以及合约的一些重要方法，受其Owner控制
- **节点所持有的外部账户：**持有部分ETH，用于在提交完成请求的事务时支付交易费用，可以在需要时将其账户持有资金提出。
- **节点web管理界面的登录账户：**作用如图

![](https://lh3.googleusercontent.com/M4v_XLXAX7x7DDlJ98q9vWUT2JuMvS3AzghGBATnS5d69bkANnadKD9L_buKoFO0BWxrddOBFG8-OQLabHuBgofYKJUIkTHICwtJw75vvVZLkZNtOWPPFensQzLNfmh0SWNcuTCf)