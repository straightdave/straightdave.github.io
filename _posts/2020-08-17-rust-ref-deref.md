---
layout: post
title:  "引用和溯源"
date:   2020-08-17 22:10:10 +0800
categories: rust
comments: true
---

* [引用是一种数据结构](#引用是一种数据结构)

<br>
> 引用（_referencing_）和溯源（_dereferencing_）是铁锈语言学习和应用中相当常见的概念，
> 有必要深入研究。有了深层次的理解才能做到使用时胸有成竹、得心应手。

为了不转移数据的“所有权”，铁锈创造了“借用”的概念：
```rust
let a = 10;
let b = &a;
```

在变量前面加上 `&` 符号，便创建了一个对于该变量的引用。
此处变量 b 就是 a 的引用，它可以 “借用” a 的值，但是又不夺走 a 对值的所有权。

> 引用符号 `&` 没必要紧贴着变量名。`&   a` 也是合法的引用操作，只是相当不美观。

## 引用是一种数据结构

上述代码中，b 的类型是一个称为“引用”的特殊数据结构。
该数据结构在内存中的造型类似于指针，是默认分配在 **栈** 上的一小段空间：

<img src="/assets/img/rust-reference.png">

引用类型内部除了目标数据的起始地址之外，应该还记录了目标数据的类型、引用方式等信息。
由于内存访问安全因素，铁锈不会直接让程序员查看和操作这些危险的信息，因此这个数据结构对于程序员来说比较抽象。
但请记住，它和 `String`，`Vec<T>` 一样，是铁锈内建的数据结构。

这里变量 b 的类型是 `&i32`，可以读作“32位整型数值的（只读）引用”。
尽管不是基本数值类型，但它们之间的赋值操作是数据“复制”：
```rust
let a = 10;
let b = &a;
let c = b;
println!("c={}", c); // => 10
println!("b={}", b); // => 10
```

看起来如此！只读引用 b 在将值赋给 c 之后仍然可用，说明没有转移所有权，而是把自己的数据原样复制一份给 c。

那么如果 b 是一个可写引用呢？
```rust
let mut a = 10;
let b = &mut a;
let c = b;
println!("c={}", c);
println!("b={}", b);
```
> 这里 b 的类型是 `&mut i32`，可以读作“32位整型数值的可写引用”。

这里的赋值行为终于导致了“所有权已经转移”的错误：
```
error[E0382]: borrow of moved value: `b`
  --> src/main.rs:10:22
   |
7  |     let b = &mut a;
   |         - move occurs because `b` has type `&mut i32`, which does not implement the `Copy` trait
8  |     let c = b;
   |             - value moved here
9  |     println!("c={}", c);
10 |     println!("b={}", b);
   |                      ^ value borrowed here after move
```

仔细看错误信息，我们可以得知，可写引用类型之所以发生了所有权转移，是因为它 _没有实现 Copy 特性_。
换言之，只读引用类型实现了 _Copy 特性_，因而在赋值时，将自身的值复制了一份，而没有转移数值的所有权。



（未完待续）


<br>
<hr>

<div id="disqus_thread"></div>
<script>
(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = 'https://straightdave-github-io.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
