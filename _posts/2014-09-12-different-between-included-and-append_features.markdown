---
layout: post
title: "浅析Module类中included和append_features两个回调方法的区别"
date: 2014-09-12 13:30:31
category: ruby
---

因为要讲Concern，所以这几天在看rails有关这部分的源码，结果发现了一个之前没见过的方法：append_features。随手翻了下[ruby-doc上的说明](http://ruby-doc.org/core-2.1.2/Module.html#method-i-append_features)，说明如下：

> When this module is included in another, Ruby calls append_features in this module, passing it the receiving module in mod. Ruby’s default implementation is to add the constants, methods, and module variables of this module to mod if this module has not already been added to mod or one of its ancestors. See also Module#include.

看完了感觉似乎自己明白了点，但是又好像没明白这东西到底跟喜闻乐见的included有啥区别 OTL

于是自己写了俩Module测试了下，似乎搞明白咋回事了。

```ruby

#覆盖掉Module类的两个同名方法
module Guest
  def self.included(target)
    puts 'method included has been triggered'
    result = target.instance_methods.include?(:new_method)
    puts "has new_method been included?: #{result}"
  end

  def self.append_features(target)
    puts 'method append_features has been triggered'
    result = target.instance_methods.include?(:new_method)
    puts "has new_method been included?: #{result}"
  end

  def new_method
  end
end

class Host
  include Guest
end
```

运行 Host.new 后，会得到如下结果：

```bash
method append_features has been triggered
has new_method been included?: false
method included has been triggered
has new_method been included?: false
```

可以发现，append_features原来是运行于included之前的，并且被我们覆盖掉之后，混入不起作用了……

修改一下 append_features 方法，改为：

```ruby
  def self.append_features(target)
    puts 'method append_features has been triggered'
    result = target.instance_methods.include?(:new_method)
    puts "has new_method been included?: #{result}"
    #增加调用Module类的append_features
    super
    #看看调用后会发生什么
    result = target.instance_methods.include?(:new_method)
    puts "has new_method been included?: #{result}"
  end
```

再次运行，可得到如下结果:

```bash
method append_features has been triggered
has new_method been included?: false
has new_method been included?: true
method included has been triggered
has new_method been included?: true
```

可以看到 Guest 模块的 new_method 方法已经被成功混入 Host 类了。

从上面试验可以得出结论：

1. 如api文档里所描述，append_features实际上是在模块被混入时调用，可以覆盖这一方法来修改混入时的行为逻辑；而included则是在模块被成功混入目标类之后才会执行的回调，无法左右混入时的行为。
2. 通过调用super方法，append_features某种意义上可以替代included的功能……？
