# LLVM 入门笔记

## 环境搭建
WSL2 + Ubuntu22 + llvm-15

从源码构建 llvm：
不久前试了一次，然后笔记本电脑爆炸了，嘻嘻。目前还不太需要直接修改 llvm 源码，所以先搁置一下。

从 apt 源安装llvm-15：
```sh
wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key | sudo apt-key add -
sudo apt-add-repository "deb http://apt.llvm.org/jammy/ llvm-toolchain-jammy-15 main"
sudo apt-get update
sudo apt-get install -y llvm-15 llvm-15-dev llvm-15-tools clang-15
```
过程中可能会遇见 zlib 库找不到，去官网下一个编译就行。

其余需要的依赖：(括号内为我配置的版本)

- CMake 3.13.4 或更高版本 (3.25.1)
- 支持 C++17 的 C++ 编译器 (11.3.0)
- 可选：[lit](https://llvm.org/docs/CommandGuide/lit.html) (15.0.6)
## 一些原理

这里选择参考 [llvm-tutor](https://github.com/banach-space/llvm-tutor/) + SG爹的[博客](https://www.cnblogs.com/Here-is-SG/p/16708170.html)进行一个入门的学习。

### LLVM 架构

在没有学 LLVM 之前，只知道它是编译器后端，但具体什么是后端，LLVM 在编译过程中有什么作用便一无所知了。

LLVM（Low Level Virtual Machine）是一种模块化的编译器框架，在设计时将整个编译层次划分得非常清楚：

-   **Frontend:前端**  词法分析、语法分析、语义分析、生成中间代码
-   **Optimizer:优化器**  中间代码优化
-   **Backend:后端**  生成机器码

![](https://image.silente.top/img/3008243-b517c768f5a97607.webp)

其特点有：
-   不同的前端后端使用统一的中间代码 LLVM Intermediate Representation (LLVM IR)
-   如果需要支持一种新的编程语言，那么只需要实现一个新的前端
-   如果需要支持一种新的硬件设备，那么只需要实现一个新的后端
-   优化阶段是一个通用的阶段，它针对的是统一的LLVM IR，不论是支持新的编程语言，还是支持新的硬件设备，都不需要对优化阶段做修改

对于每个不同层次的应用也各有不同。对于前端层次，可以做语法树解析，语言转换，创造新语言之类的；对于优化器层次，可以来做代码分析，优化或者混淆。

这里有一个点就是为什么在静态分析的时候使用的是 IR 而不是 AST呢？原因来自 Static Program Analysis Book：

-   AST 是 high-level 且接近语法结构的，而 IR 是 low-level 且接近机器代码的。
-   AST 是依赖于语言的，IR 通常是独立于语言的：三地址码会被分析器重点关注，因为可以将各种前端语言统一翻译成同一种 IR 再加以优化。
-   AST 适合快速类型检查，IR 的结构更加紧凑和统一：在 AST 中包含了很多非终结符所占用的结点（body, assign 等），而 IR 中不会需要到这些信息。
-   AST 缺少控制流信息，IR 包含了控制流信息：AST 中只是有结点表明了这是一个 do-while 结构，但是无法看出控制流信息；而 IR 中的 goto 等信息可以轻易看出控制流。

因此 IR 更适合作为静态分析的基础。

### Static Single Assignment
From SG
> 在编译器设计中，静态单赋值（Static Single Assignment, SSA），是 IR 的一种属性。简单来说，SSA 的特点是：在程序中一个变量仅能有一条赋值语句。静态的意思就是在整个静态程序，而非动态执行中的程序里，一个变量只能出现在一条赋值语句里。

为了解决这种变量引用混乱的问题，便使用一个合并操作符$\phi$（phi-function），根据控制流的信息确定使用哪个变量。

更多相关知识点：

- [中间表示](https://spa-book.pblo.gq/ch1/2intermediaterepresentation)
- [官方文档](https://llvm.org/docs/)
![](https://image.silente.top/img/1596887-20220927134430112-1076343907.png)
### IR 和 Pass

**LLVM IR** 是 **LLVM Intermediate Representation**，它是一种 low-level languange，由前端从源代码翻译而来。对于 C/C++来说， `clang` 便是 LLVM 架构的前端；对于 Rust 来说，`rustc` 是 LLVM 架构的前端。

- [LLVM IR opcodes](https://github.com/llvm/llvm-project/blob/release/15.x/llvm/lib/IR/Instruction.cpp#L347-L426) 
- [LLVM指令集入门](https://blog.csdn.net/qq_37206105/article/details/115274241)

**LLVM Pass** 是 LLVM 提供给用户用来干预代码优化过程的框架。使我们能够调用接口对于中间代码构建自己的想法。

## HelloWorld

来自 llvm-tutor，HelloWorld Pass 作用是遍历函数名和参数个数并输出。
```cpp
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"
#include "llvm/Support/raw_ostream.h"

using namespace llvm;

namespace {
void visitor(Function &F) {
    errs() << "(llvm-tutor) Hello from: "<< F.getName() << "\n";
    errs() << "(llvm-tutor)   number of arguments: " << F.arg_size() << "\n";
}

struct HelloWorld : PassInfoMixin<HelloWorld> {
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &) {
    visitor(F);
    return PreservedAnalyses::all();
  }
  static bool isRequired() { return true; }
};
}

llvm::PassPluginLibraryInfo getHelloWorldPluginInfo() {
  return {LLVM_PLUGIN_API_VERSION, "HelloWorld", LLVM_VERSION_STRING,
          [](PassBuilder &PB) {
            PB.registerPipelineParsingCallback(
                [](StringRef Name, FunctionPassManager &FPM,
                   ArrayRef<PassBuilder::PipelineElement>) {
                  if (Name == "hello-world") {
                    FPM.addPass(HelloWorld());
                    return true;
                  }
                  return false;
                });
          }};
}

extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
  return getHelloWorldPluginInfo();
}
```

```sh
# 构建 pass
mkdir build && cd build
cmake -DLT_LLVM_INSTALL_DIR=$LLVM_DIR HelloWorld/ && make

# 产出待操作的中间文件
clang -emit-llvm inputs/input_for_hello.c -S -o inputs/input_for_hello.ll

# opt 加载 pass 对 IR 进行操作
$LLVM_DIR/bin/opt -load-pass-plugin --disable-output ./build/libHelloWorld.so -passes=hello-world inputs/input_for_hello.ll
```

对于现代 llvm 的 pass 加载有两个 PM (Pass Manager)可供选择，分别是 legacy 和 new，我更倾向于学新不学旧，所以就 New Pass Manager 的写法进行学习了hh。文章参考：[LLVM’s New Pass Manager](https://blog.llvm.org/posts/2021-03-26-the-new-pass-manager/)

NewPM 中，对于我们自定义的 Pass 类需要继承自 `PassInfoMixin` 或者其子类，然后实现 `run` 方法。默认情况下，Pass 插件是动态编译和链接的，所以需要提供一个接口：
```cpp
extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
  ...
}
```

当然直接可以调用静态链接所需要的函数进行注册。
```cpp
llvm::PassPluginLibraryInfo get##Name##PluginInfo() {
	...
}
```

对于 example 来说便是向 pipeline parse 注册了一个 FunctionPassManager 的回调函数，若 passes 参数中有 `hello-world` 便会进入这一个 pass 中。
## 参考
- https://www.jianshu.com/p/1367dad95445
- https://spa-book.pblo.gq/
- https://llvm.org/docs/WritingAnLLVMPass.html