# Lifetime 问题

### `serde_json::from_slice` 

```rust
async fn get<'a, T, D>(&self, url: Url, params: &T) -> Result<D>
where
	T: Serialize + ?Sized, D: Deserialize<'a> {
	let response = self.inner.get(url)
		.query(params)
		.send()
		.await?;
	let data = response.bytes().await?;
	match serde_json::from_slice::<D>(&data[..]) {
		Ok(content) => Ok(content),
		Err(e) => panic!("{}", e)
	}
}
```
![](https://cdn.silente.top/img/p1.png)

解决：[stackoverflow](https://stackoverflow.com/questions/43554679/how-to-fix-lifetime-error-when-function-returns-a-serde-deserialize-type) 中提到有个 [serde issue891](https://github.com/serde-rs/serde/issues/891)，使用 `DeserializeOwned` 替换 `Deserialize<'a>`。

这是一个很典型的 lifetime 问题，stackoverflow 下面的回答说的比较清楚了，使用 `D: Deserialize<'a>` 意味着存在生命周期的限制，由调用者决定，而前面所生成的 `data` 会在该函数作用域结束后被回收，并不满足这个周期条件；而使用 `DeserializeOwned`，表示由被调用者决定，可以满足任何生命周期条件。原文中一句话很关键："Usually this is because the data that is being deserialized from is going to be thrown away before the function returns, so T must not be allowed to borrow from it." 

除了用 `DeserializeOwned` 表示任意生命周期，也可以使用 `for<'a> D: Deserialize<'a>`   [Higher-Rank Trait Bounds (HRTBs)](https://doc.rust-lang.org/nomicon/hrtb.html#higher-rank-trait-bounds-hrtbs) 限定。

[文档-Trait bounds](https://serde.rs/lifetimes.html#trait-bounds)


