---
layout: post
title:  "铁锈写个单链表都是地狱级硬核操作（三）"
date:   2020-08-14 12:10:56 +0800
categories: Rust
comments: true
---

<img src="/assets/img/haha_meme.jpg">

泪奔，终于写出一个版本。

这个版本遇到最大的问题是，用一个引用来遍历各个节点的 `next`，最终可以指向最后一个非空值节点的 `next` （该 `next` 指向链表最末的空值节点）。一开始使用只读引用，最后只能找到所需要的节点，但无法修改它。

随后把这个引用改为可写引用，但是在使用 `while let` 的时候，已经借用过它，则后期给它赋新值时，不能再次借用：
```rust
while let Node::Valued { val, next } = cur.as_mut() { // 第一次可写借用
    // ... 省略 ...
}
mem::replace(cur, new_node); // 作为 mem::replace() 第一个参数，是另一次可写借用
```
最后不得已用一个计数器来，帮助将整个过程放在 `while let` 内部进行。如果是最末一个 `next`，就地操作 `next` 而非游标变量，因为游标变量值的所有权转移到内部这个 `next` 变量中。

<br>
完整代码 👇
<br>
<script src="https://gist.github.com/straightdave/186eb35c92b25e14fc6ce5fa8d68a250.js"></script>


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
