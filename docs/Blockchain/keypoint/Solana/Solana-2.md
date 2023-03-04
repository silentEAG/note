---
tags:
  - TODO
---

# Solana 深入

## 数据的序列化

Solana 集群存储可以被认为是一个单一的堆。Solana 上的智能合约（Solana 术语中的“程序”）都有自己的一部分堆。

[Borsh](https://github.com/near/borsh-rs)



## Anchor 框架

```sh
sudo apt-get update && sudo apt-get upgrade && sudo apt-get install -y pkg-config build-essential libudev-dev libssl-dev
# Anchor version manager
cargo install --git https://github.com/coral-xyz/anchor avm --locked --force
avm install latest
avm use latest
```

## BPF 字节码