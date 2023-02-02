# Yubikey 与 GPG
## 前言

在10月份 yubico 跟 cloudflare 来了次联动打折，之前其实是并不知道有这东西的，然后感觉周围好多人都在刷这小玩意，自己看了看感觉还不错而且确实是骨折价，可以拿来玩玩，然后就跟朋友一起买了 4 枚 Yubikey 5C NFC（为什么一人 2 枚呢，因为怕 1 枚丢了就寄了emmm）。

Yubikey 下单后需要转运，我们选择的是转运中国，我看别的 blog 也有用顺丰国际的。大概流程就是让官方寄到日本的转运中国仓库，然后再过海关到我们手上。由于怕价值超过 1000 RMB 难过海关所以又做了次拆包操作，这么折合下来每枚的单价是 146.25 RMB，相比于国内还是超级划算的。

初次接触这类 security card，感觉功能挺多的，慢慢开始摸索一下呗。

## GPG

### 什么是 GPG

[PGP (Pretty Good Privacy)](https://zh.wikipedia.org/wiki/PGP)，这是一个非对称加密协议，而 GPG 是开源社区搞出了一个遵循此标准的免费实现，其用途主要有：

|  缩写   | 能力  | 用途  |
|  ----  | ----  | ----  |
| [C]  | Certificating | 认证其他密钥，比如签署一个子密钥。 |
| [S]  | Signing | 签名。证明某数据没有被篡改。 |
| [A]  | Authenticating | 认证。比如 SSH 登录。 |
| [E]  | Encrypting | 加密。 |

GPG 比较特殊的一点设计便是拥有子密钥的概念以及去中心化无CA。子密钥能够更加灵活的使用信任链，而无CA则表明我们需要依靠 key server 去手动分发公钥。

### Generate GPG

首先搞搞 GPG，下载[gnupg](http://www.gnupg.org/)。

```sh
# 生成密钥
> gpg --full-generate-key
# 进入密钥的管理模式
> gpg --expert --edit-key <密钥指纹>
# 添加子钥
gpg> addkey
```

我生成一个主密钥后用这个主密钥又添加了三个子钥，一个签名一个加密还有一个来认证，主密钥设置永不过期，子钥都是一年的时限。为了兼容旧设备，所以子钥都采用了 RSA，而不是默认的 ECC。

这里也有一个有趣的点是 gpg 生成 key 会给你默认生成一个用于加密且永不过期的[子钥](https://serverfault.com/questions/397973/gpg-why-am-i-encrypting-with-subkey-instead-of-primary-key)。

从安全角度，主密钥应当仅用于签署子密钥，创建后应该离线妥善保存。而日常工作则使用子密钥。

```sh
> gpg --list-keys
pub   ed25519 2023-01-15 [SC]
      6E1F5C304BA99EF54BD864E00A64925F981F851D
uid           [ultimate] SilentE <mail>
sub   cv25519 2023-01-15 [E]
sub   rsa4096 2023-01-15 [S] [expires: 2024-01-15]
sub   rsa4096 2023-01-15 [E] [expires: 2024-01-15]
sub   rsa4096 2023-01-15 [A] [expires: 2024-01-15]
```

然后再备份密钥和公钥：
```sh
> gpg --armor --export-secret-keys --export-options backup -o ./backup.gpg <id>
> gpg --armor -o ./public.gpg --export <id>
```

导入密钥使用：
```sh
> gpg --import --import-options restore ./backup.gpg
```

如果发生私钥泄漏，我们需要有一个吊销证书来吊销它，吊销证书一旦生成，**可以在没有私钥、无需密码的情况下** 声明一个私钥作废。

```sh
> gpg --output ./revoke.asc --gen-revoke <id>
```

请妥善保存以上文件。

### Add to Yubikey

```sh
# 读取卡状态
> gpg --card-status
# 进入 key 编辑状态
> gpg --expert --edit-key <id>
# 选取第x个 key 进行操作
> key <x>
# 导入到 card
> keytocard
```

导入实际是一次移动，之后主机的子密钥信息便是一个指向 yubikey 的指针。这里有一个更安全的做法是先导出子密钥信息，然后导出整个密钥并删除，再添加进子密钥，主密钥便可以离线保存。

设置每次调用卡内私钥都需要触摸 Yubikey：
```sh
.\ykman openpgp keys set-touch SIG FIXED  
.\ykman openpgp keys set-touch ENC FIXED  
.\ykman openpgp keys set-touch AUT FIXED  
.\ykman openpgp keys set-touch ATT FIXED
```

如果在某一步搞砸了或者忘记 PIN 了可以输入如下进行 reset。
```sh
> .\ykman openpgp reset
```

### Git commit with sign

```sh
> git config --global gpg.program "path/to/gpg.exe"
> git config --global user.signingkey [密钥ID]  
> git config --global commit.gpgsign true  
> git config --global user.email [邮箱]  
> git config --global user.name [github]
```

![[verify.png]]

## 其他

浅浅玩了下 GPG，其他功能看什么时候能用到再慢慢学吧hh

## 参考

- [别人整理的 Guide](https://github.com/drduh/YubiKey-Guide)
- [详解 Yubikey 5 NFC 的工作原理](https://bitbili.net/yubikey_5_nfc_functions.html)
- [玩转Yubikey(GPG)](https://acha666.cn/2021/07/07/%E7%8E%A9%E8%BD%ACYubikey)
- [Yubikey starting gpg](https://chenhe.me/post/yubikey-starting-gpg)
- [GPG概念](https://zhuanlan.zhihu.com/p/137801979)