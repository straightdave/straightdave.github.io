---
layout: post
title:  "控制 Elixir 模块的行为"
date:   2021-03-26 12:10:10 +0800
categories: elixir
comments: true
---

模块（Module）是 Elixir 的代码复用单元。一个简单的例子：

```elixir
defmodule MyLib do
  def hello(name) do
    "hello #{name}"
  end
end
```

`MyLib.hello("world")` ，它的行为就只是返回 hello world。

可以通过函数参数调整行为：

```elixir
def hello(name, say_hi_loudly?) do
  if say_hi_loudly? do
		"hello #{name} !!!!!"
	else
		"hello #{name}"
	end
end
```

但是每次执行都附上参数 say_hi_loudly 显得比较麻烦。

Elixir 可以定义模块级别的属性，函数代码可以访问这些属性，根据情况改变行为：

```elixir
defmodule MyLib do
  @say_hi_loudly true

  def hello(name) do
		if @say_hi_loudly do
			"hello #{name} !!!!!"
		else
			"hello #{name}"
		end
  end
end
```

但是 Elixir 的模块属性只能是常量，其本质上是模块的静态标签。它有如下问题：

问题一：它只能在模块定义时设置。这又导致两个问题：

1. 对所有调用方只能采用同一种行为；
2. 如果模块来自第三方库，则无法轻易修改其源码。

问题二：模块加载后，它的属性值无法改变。因此模块的行为无法在加载后动态的改变。

这两个问题是递进的需求。但问题二的解决方案相对简单一些。

### 问题一

解决问题一，可以把指导行为的状态 “注入” 到宿主代码，用宿主代码的状态来指导模块行为。这样模块被载入不同的宿主代码，可以通过宿主代码不同的属性定义做出不同的行为。

首先，利用 `__using__` 宏可以在宿主代码使用 use 方式载入模块的时候注入属性 `@say_hi_loudly` ：

```elixir
defmodule MyLib do
  defmacro __using__(opts) do
		loudly? = Keyword.get(opts, :loudly, false)
		quote do
			@say_hi_loudly unquote(loudly?)
    end
  end
end
```

宿主代码通过 use 载入，并且提供用来指导模块行为的参数：

```elixir
defmodule A do
	use MyLib, say_hi_loudly: true

  ...
end
```

编译时它会变成：

```elixir
defmodule A do
	@say_hi_loudly true

	...
end
```

（如果宿主代码在 use 的时候不传值，这个属性就用默认值 false）

然后，MyLib 的函数 hello 需要具有从宿主代码中读取属性的能力：

```elixir
def hello(name) do
	# 感知宿主模块
	# 读取宿主模块的属性 @say_hi_loudly
	# 做出相应行为
end
```

正常情况下，通过 def 定义的函数是无法感知调用者信息的（可以利用 Process 包获取调用进程以及该进程的模块信息，但这种方法比较 hacking，而且有时候不够准确）。而定义宏的时候，可以使用 `__CALLER__` 获取调用者信息。因此，我们可以做如下代码达到目的：

```elixir
defmacro hello(name) do
	caller_mod = __CALLER__.module
	loudly? = Module.get_attributes(caller_mod, :say_hi_loudly)
	if loudly? do
		"hello #{name} !!!!!"
	else
		"hello #{name}"
	end
end
```

另外值得一提，不光是 `__using__` ，其它 **宏的定义** 也都是在编译期被执行，它里面可以获取、使用编译期的数据，比如 `Module.get_attributes` ，以及 `__CALLER__`( `Macro.Env` ) 数据结构等。

至此宿主代码可以引入模块，并且为模块设置默认行为：

```elixir
defmodule A do
	use MyLib, say_hi_loudly: true

	def func do
		MyLib.hello("world")
	end
end
```

回顾上面的方案，我们让模块的宏（hello）读取之前用 `__using__` 宏注入宿主代码的属性，做出相应行为。为了让模块的行为得知谁是调用者，不得已用宏的方式定义该行为。

有种更简单的做法是，在注入属性的时候，直接注入函数。这样方式注入的函数可以直接访问注入的属性，因为他们同属于宿主代码模块：

```elixir
defmacro __using__(opts \\ []) do
	loudly = Keyword.get(opts, :say_hi_loudly, false)
	quote do
		@say_hi_loudly unquote(loudly)

		def hello(name) do
			if @say_hi_loudly do
				"hello #{name} !!!!!"
			else
				"hello #{name}"
			end
		end
	end
end
```

这种方法，唯一的区别是宿主代码使用 “自己的” 函数 hello，而不是用 MyLib 模块提供的宏。

### 问题二

从内存模型来说，状态值是需要被活的进程维护的。前面问题一的解决思路，是把纯函数库 MyLib 所需的状态变量维护在宿主代码的进程中，即宿主代码的模块属性。不过这个属性是只读的，宿主模块在引入代码时设置后就无法改变。

问题二的解决方法就是将模块转化成为一个进程。常见的方法是将模块定义为 GenServer，让其维护状态数据。调用方 —— 此时不能再称为宿主代码，因为两者都是进程 —— 通过消息与模块进行沟通，使用模块提供的行为，或者访问、修改模块的状态。

这种方案，模块的状态、行为是可以动态修改的。



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
