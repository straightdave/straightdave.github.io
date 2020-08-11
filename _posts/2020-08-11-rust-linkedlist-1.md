---
layout: post
title:  "铁锈写个单链表都是地狱级硬核操作"
date:   2020-08-11 10:10:56 +0800
categories: rust
comments: true
---

从早上 9 点写到半夜，仍然卡在编译器。用 `Rc` 都不知道绕了几圈了，各种借用关系、能否修改的错误就像打地鼠一样。


但这也说明，如果真自己写出来了，绝对说明对铁锈语言但操使基本得心应手了吧。


看，有人写了一本[小书](https://rust-unofficial.github.io/too-many-lists/)来讲这个。


成功那天再来好好讲讲这个课题的思考经历和知识总结。


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
