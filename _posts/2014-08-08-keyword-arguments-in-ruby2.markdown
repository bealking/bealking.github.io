---
layout: post
title: "使用关键字参数增加程序的可读性"
date: 2014-08-08 15:40:41
category: ruby
---

我们的代码中经常充斥着下面这样一些方法：

```ruby
  def load_file(file, format = false)
    if format
      # 一些格式化的逻辑
    end
  end
```
调用时会变成 load_file('test.csv', true)，单单只看调用的代码，你很难知道后面这个true表示什么，还需要翻文档或者源代码。

```ruby
  def spot_order(items, source, owner, by_proxy)
  end
```

除了上面的问题，还带有一大串参数，每当想要调用的时候都要打开源方法看下参数的顺序，免得放错了位置……

在ruby1.9中，也有解决方案，那就是把参数改成hash形式的：

```ruby
  def old_way(args = {})
    type = args.fetch(:type,'default')
  end

  old_way(type: 'haha') #=> 'haha'
```

但是这样也有缺陷，因为你需要写额外的代码去处理传入的参数，以保证传入的参数在方法内正常作用。而且这还是没有解决第一个例子里的问题，因为有时候你还是要去看方法内部的代码以弄清楚到底可以传入那些参数，以及参数名称。

为了解决这些问题，ruby2中正式引入了关键字参数（keword argument）。

使用关键字参数改写刚才的方法：

```ruby
    # 关键字参数
  def new_way(type: 'default')
    type
  end

  new_way #=> 'default'
  new_way(type: 'haha') #=> 'haha'
```

关键字参数无需关注参数的顺序：

```ruby
  def count(a: 0, b: 0, c: 0)
    a + b -c
  end

  count(a:1, b:2, c:3) #=> 0
  # 普通参数形式，你绝对不可以这样做
  count(b:2, c:3, a:1) #=> 0
```

ruby2.1中对关键字参数进行了加强，可以省略指定默认值来强制要求必须传入参数值：

```ruby
  def new_way2(type:)
    type
  end

  new_way #=> ArgumentError: missing keyword: type
```

## 结语

1. 在一些复杂的方法上使用关键字参数可以增加代码的可读性。
2. 一些版本迭代很快的方法，使用关键字参数可以提高可维护性。设想你有一个多态方法，在100个文件中有调用，此时如果你不得不变动参数的顺序，你会不会变得抓狂呢？
3. 提高开发效率，你不需要经常翻阅文档或者源码去查询参数的顺序，只要简单记得参数的名称即可
4. 关键字参数的缺点是，调用时需要使用符号的形式，增加了一定量的代码书写
