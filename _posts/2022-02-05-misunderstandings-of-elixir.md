---
layout: post
title:  "对 Elixir 的误解"
date:   2022-02-05 12:10:10 +0800
categories: elixir
comments: true
---

## 因为 Elixir 具有 fault tolerance 的特性，我们可以放手 let it crash？


默认情况下，supervisor 管理的进程发生崩溃后，supervisor 会对其进行重启。
那么只要系统中有 supervisor 保护，系统永远不会停止运作吗？
并不是。

```elixir
children = [
  Hello.Worker
]

opts = [strategy: :one_for_one, name: Hello.Supervisor]
Supervisor.start_link(children, opts)
```

上面是最常见的一段 Elixir 代码，常见于 `application.ex` 主文件。该 Application 主进程启动了一个 supervisor。该 supervisor 携带了 worker 子进程。启动时应用了一种策略（`strategy: :one_for_one`），并且给 supervisor 注册了名字（`name: Hello.Supervisor`）。

然而，启动 supervisor 还有两个重要选项未被提及：
* `max_restarts`: 在一段时间内允许某个子进程重启的最大次数，默认是 3（次）；
* `max_seconds`: 就是上面所说的 “一段时间”，默认是 5（秒）。

默认情况下，如果某个子进程在 5 秒内因故崩溃超过 3 次，supervisor 将不再对其重启，而任由其崩溃。子进程奔溃后，即又导致该 supervisor 自身崩溃（如果 supervisor 调用 `start_link` 启动子进程）。而且在上述代码中，启动 supervisor 的方式也是 `start_link`，所以 supervisor 崩溃后，也会导致执行该代码的进程崩溃。

因此，即使对整个系统进行了优良的设计，用 supervisor 树隔离了子系统，但依然会在某个情形下，整个系统停止工作。一旦系统停止工作，即使之后触发故障的问题消失，系统也将无法自动回复。

若干处理方案或思路：
* Supervisor 完全不用操心去重启子进程（将 restart 策略设为 `:temporary`）。但这种方法不能应用于所有情况，有的业务需求无法将子进程视为一次性、失败后可以安全丢弃的操作。
* 将 `max_restarts` 和 `max_seconds` 调整到足够大。如果可以预见某种故障发生频率，这种方法倒是可以考虑。

或者不要依赖 supervisor 的魔法，而给进程代码提供更好的防御：
* 处理可预期的错误：
  - 比如小心使用第三方带有 `!` 后缀的函数，尽可能用不带 `!` 号的版本（注意有些第三方函数使用了会抛出错误的代码，但却没有用 `!` 标记出来），然后匹配其不同的返回模式
  - 检查是不是为所有可能的模式都提供了对应的匹配？
  - 检查那些隐蔽的能够抛出错误的地方，比如 `GenServer.call` 调用超时等
* 给可疑代码加上 `rescue` 或者 `try/catch`
* 调用外部系统接口时，考虑套上 circuit-breaker：错误超过某个频率就暂时中断该调用，一段时间后再恢复。

## 结论

* 即使有 supervisor 的隔离保护，某个进程的崩溃依然可以导致整个系统停止工作。
* 在写需要长期稳定运行系统的时候，不要盲目笃信 “let it crash”，而是仍然要小心翼翼去发现和处理可能触发异常的代码。



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
