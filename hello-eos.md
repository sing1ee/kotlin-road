# 非典型EOS开发入门

#### 非典型
EOS上线稳定运行一段时间了，最近也是成为众多博彩的DAPP首选公链。但我们这篇文章不讲区块链，也不讲去中心化，也不讲币，只从技术的角度，从使用一个类似SDK的工具的角度，来一个入门之旅。毕竟EOS蛮成熟了。

> 注：以下操作都在`Mac`上进行

#### 安装

安装方法有比较简单，和半年前不可同日而语，那时候编译通过也是比较困难的事情。
首先通过`brew`安装。

```cpp
// install
brew tap eosio/eosio
brew install eosio
// uninstall
brew remove eosio
```

安装这里，用`brew`是一种快捷的方法，但是对学习无益。而且，对于`brew`的安装，似乎安装的不完整，我没有找到`eosiocpp`等命令。从源码`build`，是有的。过程略复杂，依赖比较多，需要配置一些环境变量，大家有问题可以留言讨论，Github里提一个`issue`，也OK。

#### 命令解析与基本操作

EOS安装完毕主要有三个命令：

- `nodeos`是节点程序，一系列可配置的插件，包括生产区块的功能，下文有介绍，可以用来作为本地的开发测试环境
- `cleos`负责和区块链以及钱包，进行命令行方式的交互
- `keosd`用来安全的存储钱包中的keys

三者之间的关系如下：
![582e059-411_DevRelations_NodeosGraphic_Option3](582e059-411_DevRelations_NodeosGraphic_Option3.png)

#### 单节点测试网络

本地开发需要一个环境，在安装好eos的基础之上，我们运行单节点的测试网络。

运行如下命令：

```cpp
nodeos -e -p eosio --plugin eosio::chain_api_plugin --plugin eosio::history_api_plugin
```
很快就会有区块产生，大部分的交易数都是0：

```cpp
info  2018-10-25T13:38:25.505 thread-0  producer_plugin.cpp:1490      produce_block        ] Produced block 0000a44aaf490a1d... #42058 @ 2018-10-25T13:38:25.500 signed by eosio [trxs: 0, lib: 42057, confirmed: 0]
info  2018-10-25T13:38:26.003 thread-0  producer_plugin.cpp:1490      produce_block        ] Produced block 0000a44b95765523... #42059 @ 2018-10-25T13:38:26.000 signed by eosio [trxs: 0, lib: 42058, confirmed: 0]
info  2018-10-25T13:38:26.502 thread-0  producer_plugin.cpp:1490      produce_block        ] Produced block 0000a44c3473bb8f... #42060 @ 2018-10-25T13:38:26.500 signed by eosio [trxs: 0, lib: 42059, confirmed: 0]
info  2018-10-25T13:38:27.000 thread-0  producer_plugin.cpp:1490      produce_block        ] Produced block 0000a44de15a88ef... #42061 @ 2018-10-25T13:38:27.000 signed by eosio [trxs: 0, lib: 42060, confirmed: 0]
info  2018-10-25T13:38:27.504 thread-0  producer_plugin.cpp:1490      produce_block        ] Produced block 0000a44e31032a7f... #42062 @ 2018-10-25T13:38:27.500 signed by eosio [trxs: 0, lib: 42061, confirmed: 0]
```
如上输出，则意味着节点启动成功，运行着一个叫做`eosio`的区块生产者。下图展示的是这个单节点是如何工作的。


![60539b3-Single-Host-Single-Node-Testnet](60539b3-Single-Host-Single-Node-Testnet.png)

启动之后，默认的配置文件存放位置为：

1. Mac

	```cpp
	~/Library/Application\ Support/eosio/nodeos/config
	```
2. Linux
	
	```cpp
	~/.local/share/eosio/nodeos/config
	```
同时，我们也可以使用`--config-dir`，在命令行执行配置目录，但要注意，当通过命令行制定，需要将`genesis.json`文件放到自定义的配置目录中。

同样，也可以制定数据目录，如下:

1. Mac

	```cpp
	~/Library/Application\ Support/eosio/nodeos/data
	```
2. Linux
	
	```cpp
	~/.local/share/eosio/nodeos/data
	```
也可以通过命令行来指定，参数为：`--data-dir`。

以上，节点启动成功，可以通过如下的命令检查：

```cpp
curl --request POST --url http://127.0.0.1:8888/v1/chain/get_info | python -m json.tool
```
输出为：

```cpp
{
    "block_cpu_limit": 199900,
    "block_net_limit": 1048576,
    "chain_id": "cf057bbfb72640471fd910bcb67639c22df9f92470936cddc1ade0e2f2e7dc4f",
    "head_block_id": "0000a82fa75500a7d2bc8be4d45c108c4e503600367259f349a54699c08eb2d0",
    "head_block_num": 43055,
    "head_block_producer": "eosio",
    "head_block_time": "2018-10-25T13:46:44.000",
    "last_irreversible_block_id": "0000a82e536d530d8d7e8912ff996bc42ee7dd1d3f8b5e13bfa2a1773d52bef5",
    "last_irreversible_block_num": 43054,
    "server_version": "f9a3d023",
    "server_version_string": "v1.4.1-dirty",
    "virtual_block_cpu_limit": 200000000,
    "virtual_block_net_limit": 1048576000
}
```
上面清晰的可以看到节点配置的详情信息，如`server_version_string`。更多的API可以查看官方的文档。

#### 为智能合约准备

在EOS上，智能合约采用的是`WebAssembly`格式代码，可由C++, Rust, Python, Kotlin等编写编译生成，但目前C++的支持很完善。在使用C++编写完成合约代码后，通过EOSIO软件中提供的eosiocpp工具，将C++代码编译生成WASM（wasm的文本格式是后缀是wast，下文可以看到）文件和abi文件，再利用cleos工具将代码部署到链上，也就是存到块数据中。

特别说明几个概念：

- `abi` 应用程序二进制接口（ABI）是一个基于JSON的描述，介绍如何在JSON和二进制表示之间转换用户操作。ABI还描述了如何将数据库状态转换为JSON或从JSON转换。文件格式类似JSON，用来定义智能合约与EOS系统外部交互的数据接口。将cpp编译为abi：

	```cpp
	eosiocpp -g ${contract}.abi ${contract}.hpp
	```
- `wast` 任何要部署到EOSIO区块链的程序都必须编译成WASM格式。这是区块链接受的唯一格式。`.wast`是`.wasm`的文本格式；将cpp编译为WASM：

	```cpp
	eosiocpp -o ${contract}.wast ${contract}.cpp
	```

智能合约名即账号名，在上述部署合约（下面Hello EOS会介绍如何部署和测试）时就已经绑定了账号。在满足条件或被调用时，超级节点就会执行相关合约，并将执行结果的数据更新到内存数据库中，同时也会更新到块数据中。

与以太坊简单对比：

1. 合约名称与合约地址的差异：以太坊合约通过地址区分，EOS的合约名就是账号名。
2. 合约更新的差异。如果以太坊合约要更新，就是一个新的地址，所以以太坊才是真正的`智能合约`，EOS是通过账户名区分，智能合约和普通的应用类型一样，可以更新。所以这个点上，`EOS更多的是给应用提供了可靠遍历的支付方式`。我想这一点很重要。
3. 资源消耗，以太坊消耗GAS，每个操作都有手续费。EOS智能合约不需要手续费，但部署合约需要RAM，发送消息和执行合约需要消耗抵押获得的CPU和网络带宽。

接下来就动手，来一个Hello World。

#### Hello EOS

开始编写EOS智能合约，EOS智能合约是采用C++语言编写，是项目方综合考虑安全和效率。

编写智能合约可以通过命令生成模板：

```cpp
$ eosiocpp -n hello
```
当然，现阶段可以直接copy如下的代码：

```cpp
#include <eosiolib/eosio.hpp>
#include <eosiolib/print.hpp>
using namespace eosio;

class hello : public eosio::contract {
  public:
      using contract::contract;
      /// @abi action 
      void hi( account_name user ) {
         print( "Hello, World", name{user} );
      }
};
EOSIO_ABI( hello, (hi) )
```
简单说明一下代码：

```cpp
#include <eosiolib/eosio.hpp>
#include <eosiolib/print.hpp>
```
引入EOS智能合约的头文件。

```cpp
using namespace eosio;
```
使用默认的命名空间，可以自定义。

```cpp
class hello：public eosio :: contract {
```
EOS中，智能合约都要继承`eosio :: contract`。


编译EOS智能合约

```cpp
eosiocpp -o hello.wast hello.cpp
eosiocpp -g hello.abi hello.cpp
```

部署合约需要创建测试用的钱包、密钥和账户。

EOS安装启动之后，默认有一个default钱包，我们也可以通过如下命令创建

```cpp
cleos wallet create -n <钱包名字>
```
创建钱包的同时，会生成一个密码，请妥善保存，在对钱包进行一些操作的时候，需要这个密码，例如解锁钱包。我们的演示里，直接使用default钱包。

在以下的过程中，如果提示钱包已经锁定，则执行解锁的命令：

```cpp
cleos wallet unlock -n default
```
解锁名字为`default`的钱包

接下来用`cleos`创建一个密钥对：

```cpp
cleos create key
```
保存好private key和public key，后面创建账号的时候，需要用。

在创建账号之前，需要将密钥导入到钱包中：

```cpp
cleos wallet import -n scuser --private-key <your_private_key>
```

我们需要两个密钥对，重复上面的两个步骤，再生成一个密钥对，然后导入钱包。

此时通过如下命令：

```cpp
cleos wallet keys
```
可以看到刚导入的两个密钥对，展示的是public key：

```cpp
[
  "EOS62d7M3N7ZkY5gMFhGKygenKNnrAtuGrvjkv9ap3Py1seLo9DYh",
  "EOS8B2wVFnPk1TNTVLjdKgtiLMtv8r8KdHKWj3U7zciUvMyo36bnT"
]
```
大家这里看到的具体的public key，与我不同。

下面要创建账号，命令如下：

```cpp
$ cleos create account eosio myuser EOS8B2wVFnPk1TNTVLjdKgtiLMtv8r8KdHKWj3U7zciUvMyo36bnT EOS62d7M3N7ZkY5gMFhGKygenKNnrAtuGrvjkv9ap3Py1seLo9DYh
Error 3090003: Provided keys, permissions, and delays do not satisfy declared authorizations
Ensure that you have the related private keys inside your wallet and your wallet is unlocked.
```
出现问题，按照如下方法解决：

1. 打开配置文件
	1. Mac

		```cpp
		~/Library/Application\ Support/eosio/nodeos/config
		```
	2. Linux
	
		```cpp
		~/.local/share/eosio/nodeos/config
		```
2. 找到signature-provider的私钥
3. 将私钥导入到钱包中

再次尝试成功，结果如下

```cpp
executed transaction: 199fcf44e40602dff76d8231b5f11129f0fda9c15fef3b684ef4149e032bab79  200 bytes  904 us
#         eosio <= eosio::newaccount            {"creator":"eosio","name":"myuser","owner":{"threshold":1,"keys":[{"key":"EOS8B2wVFnPk1TNTVLjdKgtiLM...
```

这时执行`cleos wallet keys`，就会发现，多了一个public key：

```cpp
[
  "EOS62d7M3N7ZkY5gMFhGKygenKNnrAtuGrvjkv9ap3Py1seLo9DYh",
  "EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV",
  "EOS8B2wVFnPk1TNTVLjdKgtiLMtv8r8KdHKWj3U7zciUvMyo36bnT"
]
```

智能合约成功发布到本地测试网络

```cpp
$ cleos set contract myuser ~/Code/eos-sc/hello
```
结果如下：

```cpp
Reading WASM from /Users/zhangcheng/Code/eos-sc/hello/hello.wasm...
Publishing contract...
executed transaction: 9cb1a47cd1bfe749d0b4ab4f2e15c778f798aead1103962f4e230da8f1b83f19  1808 bytes  1701 us
#         eosio <= eosio::setcode               {"account":"myuser","vmtype":0,"vmversion":0,"code":"0061736d01000000013b0c60027f7e006000017e60027e7...
#         eosio <= eosio::setabi                {"account":"myuser","abi":"0e656f73696f3a3a6162692f312e30000102686900010475736572046e616d65010000000...
```

接下来调用合约，通过`push action`命令来执行合约方法`hi`:

```cpp
$ cleos push action myuser hi '["world"]' -p myuser
```
结果如下：

```cpp
executed transaction: a3781e50591348427eb61dd2d129d1a976977f67421da791f4a3cff6923d433c  104 bytes  588 us
#        myuser <= myuser::hi                   {"user":"world"}
```

此时，`nodeos`的日志也发生了相应的变化，明确显示有一个交易。

```cpp
info  2018-10-25T09:58:27.005 thread-0  producer_plugin.cpp:1490      produce_block        ] Produced block 00003d2df2f11915... #15661 @ 2018-10-25T09:58:27.000 signed by eosio [trxs: 1, lib: 15660, confirmed: 0]
```

至此，EOS的Hello World就结束了。

下一步，来一个高级点的应用，我们在EOS上发个币吧。
