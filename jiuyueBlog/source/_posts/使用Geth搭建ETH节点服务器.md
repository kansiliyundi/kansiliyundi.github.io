---
title: 使用Geth搭建eth钱包节点服务器(一次在ETH主网被攻击后的实验)
date: 2021-4-21 19:24:55
tags: blockChain
---
## 因
因为工作需要,我最近在ETH主网上发了ERC20的合约，上了uniswap去中心化交易所，但是在上币后发现有黑客在扫描uni的上币合约，用极低的价格第一时间购买新发售的币. 

我们当然也损失惨重，为了搞明白这其中的原理，我自己搭建了以太坊的节点用来研究黑客的攻击手法，并用nodejs + ether实现了三明治攻击。 
<!-- more --> 
搭建节点的客户端使用的是Geth，Geth是go语言编写的以太坊官方客户端，提供了以太坊各种交互功能

## 服务器配置
```
OS: CentOS SELinux 8 x64
CPU: 8 vCore
RAM: 32768 MB
Storage: 640 GB SSD
```

## 安装Geth
```
# 拉代码
git clone https://github.com/ethereum/go-ethereum.git
cd go-ethereum/
git checkout release/1.10.8
# 编译
cd go-ethereum/
make all
```
#### 配置环境变量
```
vim /etc/profile
将 export PATH=$PATH:/opt/ethereum/go-ethereum/build/bin 添加到文件最后
运行 source /etc/profile 更新配置
执行 geth version 如果有结果说明配置完成
```
### 使用Geth：
```
# 第一次使用可以加上dumpconfig 将配置文件保存下来,下次启动直接使用配置文件启动,修改参数也可以直接修改配置文件
geth --datadir /xxx/xxx/ --syncmode "fast" --mine --miner.threads=4 --miner.etherbase '0x0000' --http --http.api="admin,debug,web3,eth,txpool,personal,ethash,net"  --http.addr 0.0.0.0 --http.port 8545  --http.corsdomain "*" --http.vhosts "*"   --ws --ws.addr 0.0.0.0 --ws.port 8546 --ws.api="admin,debug,web3,eth,txpool,personal,ethash,net"  --ws.rpcprefix "/"   --ws.origins "*" --txpool.journal "/xxx/xxx/"  --verbosity 3 --maxpeers 100 --maxpendpeers 100 --allow-insecure-unlock --cache=8192 dumpconfig > "/xxx/xxx.toml"


# 之后再启动可以使用命令 --config 指定配置文件启动
# nohup表示在后台运行
# & > xxx.out 表示输出log到文件
nohup geth --config /xxx/xxx.toml & > nohup.out

```

#### 参数解释
```
# 同步数据存放的路径
--datadir /xxx/xxx/ 
# 同步模式 ("fast", "full", "snap" or "light") (默认: snap)  
# full同步全部数据,并重放区块中的交易以生成状态数据 
# fast获取全部数据但不做重放,较老的数据可能会丢失  
# snap只同步区块头，不同步区块体，也不同步状态数据
--syncmode "fast" 
# 开启挖矿模式
--mine
# 挖矿启用线程数
--miner.threads=4
# 挖矿的钱包地址(说白了挖到的ETH存到哪)
--miner.etherbase '0x0000'
# 开启HTTP服务
--http
# 开放的HTTP接口
--http.api="admin,debug,web3,eth,txpool,personal,ethash,net"
# HTTP服务监听地址
--http.addr 0.0.0.0 
# HTTP服务监听端口
--http.port 8545 
# 接受跨域请求的地址,用逗号分隔 *表示全部
--http.corsdomain "*"
# 接受请求的地址,用逗号分隔 *表示全部
--http.vhosts "*" 
# 开启WebSocket服务
--ws 
# WebSocket服务监听地址
--ws.addr 0.0.0.0
# WebSocket服务监听端口
--ws.port 8546 
# 开放的WebSocket接口
--ws.api="admin,debug,web3,eth,txpool,personal,ethash,net" 
# JSON-RPC服务的HTTP路径前缀。使用'/'服务于所有路径。
--ws.rpcprefix "/" 
# 接受来自*所有域名的请求
--ws.origins "*" 
# pendingTransactions(待交易池)数据存储路径
--txpool.journal "/root/etherChainData/"
# 日志模式 0=silent, 1=error, 2=warn, 3=info, 4=debug, 5=detail (default: 3)
--verbosity 3
# 最大连接同步节点数量（这个数量和CPU占用息息相关）
--maxpeers 100 
# 最大的挂起尝试连接数 默认0
--maxpendpeers 100
# 允许不安全的账户访问Account相关的HTTP API
--allow-insecure-unlock
# 当你使用了fast模式同步数据时，cache是必须的 1024是1G 2048是2G 根据你的内存情况决定
--cache=8192
# 保存启动参数到xxx.toml配置文件
dumpconfig > "/xxx/xxxx.toml"
```

### 后续思路解析
经过以上的步骤Geth客户端已经启动起来了，你可以开始写代码用HTTP或者WebSocket的方式连接它了

说说我的实验结果
##### 初步思路
Geth搭建起来后,只要你有uni发币合约的地址其实就可以使用web3或者ether订阅这个合约的新币发布，通过解析交易信息就能拿到新池地址然后用自己的钱包去下单，但这远远不够，因为Uni上有很多陷阱合约,只能买不能卖，我们缺乏对这个行业的理解，和对陷阱合约代码分析的能力，所以很难做到短时间就盈利 
##### 三明治攻击
但是我又想到新的方法，找到具备这些分析能力的钱包地址，跟踪他们的交易，对他们进行三明治攻击，也就是监听他们的pending交易，抢先在他们完成交易前，完成我们自己的交易，在他们卖出前，卖出我们的交易，这样他们的买入操作就被夹在我们的买入卖出之间，完成了一次三明治攻击 

刚开始确实可行，我用nodejs写了一整套的eth链全链扫描，从中筛选被攻击者，然后调用uni合约下单买入，卖出，甚至做了简单的前端页面，可以配置受害人地址，可以设置下单金额，甚至配置止盈止损，这套代码也确实赚到钱。 
##### 阻塞攻击
但是很快我发现这个路数行不通了，目前的缺点有三个，一个是我的受害人同时也是合约的攻击者，他们可能是矿池主或者其他什么，他们会批量发起交易，在一个区块内用不同的账户对同一个token购买，把一个区块占满，也叫阻塞攻击，这样可以让其他人在他们同一时间下不了单抢不了他们的赢利点，同时也拦截了三明治攻击，一个是geth需要的服务器配置很高，因为他们的阻塞攻击导致我们的很多三明治攻击不能完成，盈利远远抵不了支出。还有一个问题就是以太坊链的手续费太高了，基本gas到40左右就不是我可以承受的了
##### 结语
大概思路就是这样，区块链的世界很有意思，从监听新交易，三明治攻击，阻塞攻击，这里面套路层出不穷，而我发现的这些还仅仅是皮毛