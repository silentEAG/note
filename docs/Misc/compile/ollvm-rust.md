# 在 Rust 中使用 OLLVM


## x86_64-unknown-linux-gnu

使用 Rust 1.70.0 + llvm 16

相关 Project:

- https://github.com/rust-lang/rust
- https://github.com/rust-lang/llvm-project
- https://github.com/joaovarelas/Obfuscator-LLVM-16.0

构建依赖环境： cmake, make, ninja, clang ...

首先是编译 ollvm

```sh
# 克隆源码
$ git clone --single-branch --branch 1.70.0 --depth 1 https://github.com/rust-lang/rust rust-1.70.0
$ git clone --single-branch --branch rustc/16.0-2023-03-06 --depth 1 https://github.com/rust-lang/llvm-project llvm-16.0-2023-03-06
$ git clone --single-branch --branch llvm-16.0.0rel --recursive --depth 1 https://github.com/61bcdefg/Hikari-LLVM15 ollvm-16.0

# 应用 ollvm patch
$ git apply --reject --ignore-whitespace ../llvm.patch


# 开始构建
$ cd llvm-16.0-2023-03-06/
$ mkdir build && cd build/
$ cmake -G "Ninja" ../llvm -DCMAKE_INSTALL_PREFIX="./llvm_x64" -DCMAKE_CXX_STANDARD=17 \
  -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS="clang;lld;" -DLLVM_TARGETS_TO_BUILD="X86" \
  -DBUILD_SHARED_LIBS=ON -DLLVM_INSTALL_UTILS=ON -DLLVM_INCLUDE_TESTS=OFF -DLLVM_BUILD_TESTS=OFF \
  -DLLVM_INCLUDE_BENCHMARKS=OFF -DLLVM_BUILD_BENCHMARKS=OFF -DLLVM_INCLUDE_EXAMPLES=OFF \
  -DLLVM_ENABLE_BACKTRACES=OFF -DLLVM_BUILD_DOCS=OFF  -DBUILD_SHARED_LIBS=OFF \
  -DCMAKE_CXX_COMPILER=clang+ -DCMAKE_C_COMPILER=clang

$ cmake --build . -j$(nproc)
$ cmake --install .

# 校验
$ ./llvm_x64/bin/llvm-config  --version
$ ./llvm_x64/bin/clang --version
```

然后是准备 rustc 编译：

1. 复制 `config.example.toml` 到 `config.toml`
2. 编辑 `config.toml`
   1. `debug = false` 提速编译过程
   2. `channel = "nightly"` 编译为 nightly 版本
   3. 设置 `llvm-config = "/path/to/ollvm/bin/llvm-config"` 替换为 ollvm 的 llvm-config 路径

```sh
$ cd rust-1.70.0/
$ python x.py build
$ python x.py build tools/cargo
$ python x.py dist
$ ./build/x86_64-unknown-linux-gnu/stage1/bin/rustc --version --verbose
$ ./build/x86_64-unknown-linux-gnu/stage1-tools-bin/cargi --version
```

构建完成后使用 `rustup` 进行 `link`

```sh
rustup toolchain link ollvm-rust-1.70.0 "path/to/.../stage1/"
rustup show
```

使用：

```
cargo +ollvm-rust-1.70.0 build --release
```

具体的参数可以去看 Hiakri-LLVM 的文档。

目前使用下来有两个特别严重的问题，因此这个方案可能不适合在一般的 project 中使用。

第一个问题是 build-std 参数会失败，如果之前使用 rustup 下载了新版的 cargo，其的编译选项会以新版的为准，解决的话需要使用自己编译的 cargo 进行构建，而此时又会有很多其他问题hh。
第二个问题是使用 Hiakri 提供的参数进行混淆，如果引入了一些其他 crate，那么在编译过程中会出现内存占用爆满的情况，然后整个系统卡死。

## x86_64-pc-windows-gnu

UNFINISHED

尝试过一次，但是感觉不是很成功，还在探索中。
