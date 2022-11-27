---
layout: post
title:  "铁锈写个单链表都是地狱级硬核操作（二）"
date:   2020-08-12 10:10:56 +0800
categories: Rust
comments: true
---

在本文写作时间内，构建单链表的操作依然没有搞定。不过过程中也收获了许多知识点，很多可以说是完全新鲜的概念。记录在本文内。

构建一个静态的单链表比较简单，但其中也涉及一些基础问题：空引用，递归定义等等。

## 空引用

铁锈语言没有预定义的空引用（没有 `null` 或者 `nil`！）。比如定义一个结构体 _Person_，通过借用符号 `&` 对该类型进行引用:
```rust
struct Person {
    name: String,
    age: i32,
}
let a = Person { name: "dave", age: 18 };
let b: &Person = &a; // 引用了某 Person 值
```

可惜没有办法来表示 _Person_ 类型的空值引用：
```rust
let a_null_ref: &Person = nil; // 没有预定义的空引用
```

这样也不行：
```rust
let a_null_ref: &Person; // 可以在后期给该变量赋值，但是不能直接使用这个变量作为空值。否则编译错误：未初始化！
```

空值引用在很多场合必不可少，比如单链表的尾节点。

铁锈语言的常规操作是利用枚举类型来实现同样的效果。由于枚举在铁锈语言中得到了加强，程序员可以用匿名结构体绑定枚举值，用这种枚举值的定义来取代结构体定义，还可以捎上一个枚举值表示该类型的空值：

```rust
enum Person {
    Valued { name: String, age: i32 }, // 有值
    Nil, // 空值
}

let a = &Person::Valued { name: String::from("dave") }; // 引用某个值
let b = &Person::Nil; // 引用该类型的空值
```

## 递归定义
链表的节点就是一个递归定义的数据结构：
```
节点 = (数据，&下一个节点)
```

回顾铁锈语言的类型系统的特性：
* 所有类型默认都是分配在栈上
* 分配在栈上的类型，其占用内存大小必须编译期已知

因此如果像下方代码定义一个递归的类型，是不可以的：
```rust
struct Node {
    val: i32,
    next: Node,
}
```

因为这样的话，该结构体占用空间未知，根据节点数量，可能是无穷大。




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
