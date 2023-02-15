---
tags:
  - TODO
---

# Solana 入门笔记

## 环境配置

### 系统环境

WSL2 Ubuntu22.04，和一个科学上网环境，最好设置成 Tun 模式 （能够避免很多麻烦）。

### Rust

安装
```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

官方上写的是使用 stable 工具链，我环境一直是 nightly 目前也没遇到过大的问题。`cargo --version` 查看版本信息，`rustup toolchain` 对工具链进行操作。

但需要注意的是 rust 编译器版本更迭较快，在复现一些题目时遇到一些低版本 solana crate 包，对于大于 1.60 的 Rust 编译器会[报错失败](https://github.com/solana-labs/solana/issues/25474)。

Rust ide 现在一般都是用的 vscode + rust-analyzer 了，也可以尝试 JetBrain 家的 Rust 插件。

### Node (client 交互可选)

直接下载编译好的 tar 包，不走 apt 源
```sh
curl -o node.tar.gz https://cdn.npmmirror.com/binaries/node/latest-v18.x/node-v18.14.0-linux-x64.tar.gz
tar -zxf ./node.tar.gz
# ...
# 个人喜欢用 pnpm 来替代 npm
npm install pnpm -g
```
一些 solana 依赖:

- @solana/web3.js
- @solana/spl-token

简单尝试了一下，感觉不如直接用 Rust 写 client 交互。

### Solana Cli

```sh
sh -c "$(curl -sSfL https://release.solana.com/stable/install)"
```

然而对于一些 CPU 由于没有对 AVX2 指令集的支持，所以无法直接安装预构建包，需要从[源码编译](https://github.com/hubdev4/solana-build-from-source)。

经过测试，本地机器 i7 9代的笔电能使用 WSL/Docker 构建预编译包，但是学校服务器 `Intel(R) Xeon(R) CPU E5-2640 v4` 不知道为啥没法emmm。

从源码构建的话需要找到适配的 rust 版本，我自己构建好了一份 `x86_64_linux_unknown_gnu` 且不需要 AVX2 的 solana-1.15.1 (对应 rust 版本 1.66)，有需要的师傅可以直接下载 ([OneDrive](https://nekiro-my.sharepoint.com/:u:/g/personal/silenteag_nekiro_onmicrosoft_com/EcAq_siVKZhKhjEPbzq9Cr0B166w73soTfMBO7szluAJcw?e=I7mRWS))，或者使用 Dockerfile 自行构建：

```dockerfile
FROM rust:1.66-bullseye AS build

RUN apt-get update && apt-get install -y unzip libudev-dev pkg-config clang

RUN mkdir /build

WORKDIR /build
RUN wget -q "https://github.com/solana-labs/solana/archive/refs/tags/v1.15.1.zip" \
    && unzip ./v1.15.1.zip \
    && rm ./v1.15.1.zip

RUN ./solana-1.15.1/scripts/cargo-install-all.sh .

RUN rm -r ./solana-1.15.1
```

### 常用命令

```sh
# solana cli 更新
solana-install update
# 查看本地 solana 配置
solana config get
# 创建一个 keypair，以文件的方式默认存储在 `~/.config/solana/id.json`
# `--outfile` 参数指定 path
solana-keygen new
# 设置 keypair
solana config set -k ~/.config/solana/id.json
# 设置连接的 url
# - 主网 https://api.mainnet-beta.solana.com
# - 开发网 https://api.devnet.solana.com
# - 测试网 https://api.testnet.solana.com
solana config set --url localhost
# 给当前钱包空投 sol (限 devnet 和 testnet)
solana airdrop <num>
# 查看当前钱包余额
solana balance
# 本地测试链
solana-test-validator
# 构建 so 文件
cargo build-bpf
# 部署
solana program deploy ./target/deploy/name.so
# 从主网加载合约程序
# solana program dump -u <source cluster (m/d/t)> <address of account to fetch> <destination file name/path>
solana program dump -u m 9xQeWvG816bUx9EPjHmaT23yvVM2ZWbrrpZb9PusVFin serum_dex_v3.so
# 将合约程序加载到本地链上
solana-test-validator --bpf-program 9xQeWvG816bUx9EPjHmaT23yvVM2ZWbrrpZb9PusVFin serum_dex_v3.so --reset
```

## 概念

个人感觉官方文档将每个概念给分开来解释有好有坏，比如前面的概念有些需要看到后面才能理解，但是我也不知道怎么更好地组织他们，所以还是按照文档的概念顺序来说明。

### 账户 Account

Solana 的账户其作用是用来存放数据 (store state) 的，一共有三类账户：

- Data Account，用来存储状态
- Program Account，用来存储可执行程序
- Native Account，用来存储原生程序，如果把 Solana 比作 Linux 系统，那么这就像是[系统内核](#native-program)，提供一些基本的方法接口

可以看到存储程序的账户并没有保存状态，因此 Solana 的合约程序是 **无状态** 的，这是跟 solidity 很不同的一点。

账户数据：

|字段 | 描述 |
| --- | --- |
|data | 存储的数据 |
|executable | 判断是否是可执行程序 |
|lamports | 账户余额，1 sol = 1e9 lamports |
|owner | 账户所有者 |
|rentEpoch | 下一个需要付租金的 epoch，为 0 即表示免租金 |

在 Data Account 中又有两类：

- 系统所有账户
- 程序派生账户（PDA）

通常用户直接使用的是系统所有账户，owner 是 System Program；程序账户的 owner 是 BPF Loader；PDA 是指通过程序来生成的一类地址，它的 owner 是某个程序。

### 地址 Address
一个账户拥有一个地址，类似于文件系统，账户对应着一个文件，那么地址便是文件的路径。通常来说，地址是一个 256 位的 ed25519 公钥，但程序派生账户的地址 (PDA) 是通过 Seed 进行生成的，不会有 Keypair 的存在。

### 租金 Rent

在账户中存储数据需要花费 SOL 来维持，这部分花费的 SOL 被称作租金。如果账户中的余额大于两年租金的 SOL， 这个账户就可以被豁免付租。租金可以通过关闭账户的方式来取回。

当一个账户没有足够的余额支付租金时，这个账户会被释放，数据会被清除。

### 指令 Instruction

最小交互单元。

```rust
pub struct Instruction {
    /// Pubkey of the instruction processor that executes this instruction
    pub program_id: Pubkey,
    /// Metadata for what accounts should be passed to the instruction processor
    pub accounts: Vec<AccountMeta>,
    /// Opaque data passed to the instruction processor
    pub data: Vec<u8>,
}
```


### 交易 Transaction

组成：

- 至少一个指令 (Instruction)
- 一组待读写的账户
- 至少一个的 Signer

```rust
pub fn new_signed_with_payer<T: Signers>(
    instructions: &[Instruction],
    payer: Option<&Pubkey>,
    signing_keypairs: &T,
    recent_blockhash: Hash,
)
```

一些特点：

- 交易必须明确列出链上程序可以读取或写入的每个帐户，第一个账户始终是可读写并用于支付交易费用
- 对于每笔交易能够包含多条指令，并且在对于一些 Read-Only 的账户状态能够执行并行读操作
- 指令是最小的可执行逻辑，一个指令 fail，整个交易 fail
- 交易包括一个或多个数字签名，每个数字签名对应于交易引用的帐户地址。这些地址中的每一个都必须是 ed25519 密钥对的公钥，并且签名表示匹配私钥的持有者签名，因此“授权”交易。在这种情况下，该帐户称为签名者。帐户是否是签名者会作为帐户元数据的一部分传达给程序。然后程序可以使用该信息来做出授权决定。

### 程序 Program

Solana 中的程序便是 Solidity 中的智能合约，不同之处在于其是**无状态**的，所有的和程序交互的数据都是存储在独立的账户中，通过指令传入程序。

对于每个程序有一个单独的入口点，在 Rust 中使用 `entrypoint` 宏标注入口函数：
```rust
pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8]
) -> ProgramResult {}
```

#### Native Program


## 入门题目

### allesctf2021 secret-store

代码量相对有点多，但题目本身给了很好的交互环境，主要是让选手熟悉以下 solana。

简单分析下 cli，program 有些部分就略过了。

- **`initialize_ledger`**
  - 创建了一个 Flag Mint 账户存储 token 信息，合计有 16 个
  - 创建了一个 token account， holder 是 `flag_depot`，token 有 16 个
  - 创建了 flag 原生程序账户，名字是 `flagloader_program`
  - 创建了 store 程序账户，同时写入了字节码数据
- **`setup`**
  - 创建了一个 store 程序的派生地址，用来存储数据 secret
  - 调用 `StoreInstruction::Initialize` 
    - 携带随机生成的 secret
    - 将 `flag_depot` 对应 token 账户的 authority 设置为 store 派生地址，并在该地址 (store PDA) 写入 secret
- **`getflag`**
  - 调用 `StoreInstruction::GetFlag`
    - 一系列校验，同时检验参数的 secret 是否与 store 中的 seret 相同
    - 然后将 `flag_depot` 对应 token 账户的 authority 设置为可控的账户地址
  - 调用 `flag_program`
    - 校验 token 账户是否有余额，并验证 token owner 的 Signer 身份
    - 输出 flag

所以这题的关键是获取 secret，其实链上的数据都能被所有人看到，所以可以直接读出其数据然后调用指令控制 token 的账户。

查看 solana 链上的数据有很多方法

- Json RPC。直接向 rpc 传递 json 数据
- Solana explorer。Solana 浏览器进行查询
- 官方提供的 SDK 交互
- Solana cli

```sh
# 查看一个地址的交易记录
solana transaction-history --show-transactions --url http://localhost:1024 "address"
# 查看地址的账户信息
solana account --url http://localhost:1024 "address"
# Public Key: 
# Balance: 
# Owner: 
# Executable: 
# Rent Epoch: 
# Length: 
# ... (Data)
```

对我们而言这道题只需要找到 store 账户，然后获取其 data 就行。

![202302131507952](https://cdn.silente.top/img/202302131507952.png)

或者直接使用浏览器查看 setup 时的传参数据：

![202302131512017](https://cdn.silente.top/img/202302131512017.png)

后一个知识点就是数据的 pack/unpack。在 Rust 中对数据进行了序列化处理，由于 secret 是 u64，所以单看这一个数据类型来说，Solana 是按小字节序按字节存储，所以读取也需要这样来，这里可以使用 Python [struct](https://blog.csdn.net/qdPython/article/details/115550281) 包来帮助我们操作：

```python
import struct
struct.unpack(">Q",struct.pack("<Q", 0x7654df5eab21575e))[0]
>>> 6797939182453871734
```

然后编译一份提供的 store cli 调用 get flag

![202302131518953](https://cdn.silente.top/img/202302131518953.png)

### allesctf2021 legit-bank

- **`initialize_ledger`**
  - 创建了一个 Flag Mint 账户存储 token 信息，合计有 16 个
  - 创建了一个 token account， holder 是 `flag_depot`，token 有 16 个
  - 创建了 flag 原生程序账户，名字是 `flagloader_program`
  - 创建了 bank 程序账户，同时写入了字节码数据
  - 创建了 bank_manager 账户，拥有 100 sol
  - 


## Ref

- docs.solana.com
- solanacookbook.com
- beta.solpg.io
