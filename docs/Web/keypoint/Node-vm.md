---
tags:
  - TODO
---

# NodeJS VM 沙箱逃逸

先写一些拿沙箱外对象的方法：

- 没有其他 ctx 限制下 `this` 拿的就是一个沙箱外的对象引用。 
- 直接获取 ctx 中的引用对象，类似数字，布尔的值传递不行。
- sandbox 为 null 时可以考虑用 `arguments.callee.caller` 拿函数的调用者，沙箱对象在外部调用时显然是外部的作用域。
- 同上，也可以写 Proxy 对象来劫持属性从而触发。

## Ref
- [NodeJS VM & VM2 逃逸](https://xz.aliyun.com/t/11859)