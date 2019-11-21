# 手动编译Chainlink节点

本文我们会介绍如何手动用源码编译Chainlink节点的可执行文件。（执行环境是Ubuntu1804。）

官方推荐使用docker版本来运行Chainlink node，这会省去很多的开发环境配置的工作。如果您想要简单测试或在生产环境中使用，请按照官方文档的建议使用docker版本。本文为那些想要更灵活的配置Chainlink节点或者想要修改部分代码的开发者使用。

# 安装Golang

如果您的开发环境中已经安装了Golang，可以跳过这一步，记清自己的目录结构即可。

## 下载安装

前往[https://golang.org/dl/](https://golang.org/dl)，下载最新的golang版本。

    wget [https://dl.google.com/go/go1.12.8.linux-amd64.tar.gz](https://dl.google.com/go/go1.12.8.linux-amd64.tar.gz)

解压安装

    sudo tar -C /usr/local -xzf go1.12.8.linux-amd64.tar.gz

请根据所下载的具体Golang版本修改对应的文件名。

## 配置Golang环境变量

创建gopath目录

    cd ~ && mkdir GoPath && cd GoPath && mkdir src bin pkg

添加环境变量（以zsh为例）

    vim ~/.zshrc
    
    # add these environment variables
    export GOROOT=/usr/local/go
    export GOPATH=~/GoPath
    export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
    
    # make it work
    source ~/.zshrc

测试，在命令行中输入go，如果出现如下的输出，说明go安装成功

    ➜  ~  go
    Go is a tool for managing Go source code.
    
    Usage:
    
    	go <command> [arguments]
    
    ...

# 下载Chainlink代码

    mkdir -p $GOPATH/src/github.com/smartcontractkit 
    cd $GOPATH/src/github.com/smartcontractkit
    git clone https://github.com/smartcontractkit/chainlink.git

## 安装NodeJs, Yarn

同样如果您的环境中已经配置好了nodejs和yarn，也请跳过此步骤，如果遇到版本不兼容的问题，请根据编译时的报错提示切换对应的版本。

安装nodejs

    curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
    sudo apt-get install -y nodejs

安装yarn

    curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
    echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
    sudo apt-get update && sudo apt-get install yarn

可以输入`node -v`和`yarn -—verison`检查是否安装成功。

    ➜  chainlink git:(develop) node -v
    v10.16.3
    ➜  chainlink git:(develop) yarn -v
    1.17.3

## 编译Chainlink

进入Chainlink项目目录

    cd $GOPATH/src/github.com/smartcontractkit/chainlink

加载go依赖包

    export GO111MODULE=on
    go mod vendor

由于众所周知的原因，在国内下载某些golang的库会失败，请自行解决网络问题。

除了科学上网以外，你还可以添加国内的[goproxy](http://goproxy.io)来下载vendor包，或者使用我下载好vendor的项目仓库：[https://github.com/vlyl/chainlink](https://github.com/vlyl/chainlink) 。

加载yarn依赖包，如果出错请多次执行

    make yarndep

编译，这一步会花费较长时间，请耐心等待

    make install

编译好后，在命令行输入`chainlink -h` ，如果出现chainlink的使用提示，则说明编译成功啦。

    ➜  chainlink git:(develop) chainlink -h
    NAME:
       chainlink - CLI for Chainlink
    
    USAGE:
       chainlink [global options] command [command options] [arguments...]
    
    VERSION:
       0.6.4@fe4fc08e2c0b2055d8503e15042d1224d0ba2593
    
    COMMANDS:
       local                     Commands which are run locally
       login                     Login to remote client by creating a session cookie
    
    ...

该命令位于`$GOPATH/bin`下，如果你按照本文的指引配置的环境变量，它应该位于`~/GoPath/bin/chainlink` 。如果你没有添加该目录到环境变量中，也可以去对应的目录下来执行。

下面执行`chainlink local n`就可以启动一个chainlink节点了。注意在启动之前，请先配置好Chainlink节点运行目录和相关的环境变量。