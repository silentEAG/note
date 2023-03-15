---
tags:
  - TODO
---

# Rust Atomics and Locks

## Foreword

很久之前就想读读这本书了，但上次看完 ch1 就没继续咯。

> When writing concurrent code in Rust, you should write it the way Rust wants you to. I have since learned that although this is great advice, it leaves open the question of exactly what Rust wants. This book gives excellent answers to this question, and is thus valuable both to Rust developers wishing to learn concurrency and to developers of concurrent code in other languages who would like to learn how best to do so in Rust.

## Ch1

The `println` macro uses `std::io::Stdout::lock()` to make sure its output does not get interrupted.


### Thread Scope

可以使用 `thread::scope` 来创造一个带有作用域限制的 thread spawn。
```rust
let nums = vec![1, 2, 3];
thread::scope(|s| {
    s.spawn(|| {
        println!("len: {}", nums.len());
    });
    s.spawn(|| {
        for x in &nums {
            println!("{x}");
        }
    });
});
println!("{:?}", nums);
```

对于它所在的作用域是堵塞的。

### Thread Parking

```rust
thread::scope(|s| {
        // Consuming thread
        let t = s.spawn(|| loop {
            let item = queue.lock().unwrap().pop_front();
            if let Some(item) = item {
                dbg!(item);
            } else {
                thread::park();
            }
        });

        // Producing thread
        for i in 0.. {
            queue.lock().unwrap().push_back(i);
            t.thread().unpark();
            thread::sleep(Duration::from_secs(1));
        }
    });
```

`thread::park` 能够让该进程陷入类似睡眠的状态，CPU 不再给该进程分配时间片，直到被 `unpark`。 `thread::park_timeout` 则是在 `park` 基础上加了一个超时时间。


## Ch2 Atomic

原子操作是涉及多线程的任何事物的主要构建块。所有其他并发原语，例如互斥量和条件变量，都是使用原子操作实现的。Rust 中与原子操作相关的方法均在 `std::sync::atomic` 下。

## Ch3 Memory Ordering

happens-before relationship

### Relaxed Ordering

While atomic operations using relaxed memory ordering do not provide any happens-before relationship, they do guarantee a total modification order of each individual atomic variable. This means that all modifications of the same atomic variable happen in an order that is the same from the perspective of every single thread.

### Release and Acquire Ordering

Release and acquire memory ordering are used in a pair to form a happens-before relationship between threads. Release memory ordering applies to store operations, while Acquire memory ordering applies to load operations.

### Atomic Load and Store Operations

https://marabos.nl/atomics/

https://rustcc.cn/article?id=3259fdf2-9caa-4bf6-a835-6d58efe2f9ee

https://rustcc.cn/article?id=525e276f-1341-45c5-b727-ffc733638cc7

Memory Model: 从多处理器到高级语言 https://github.com/GHScan/TechNotes/blob/master/2017/Memory_Model.md

