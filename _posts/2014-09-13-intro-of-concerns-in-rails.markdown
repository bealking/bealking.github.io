---
layout: post
title: "rails源码赏析之Concern"
date: 2014-09-13 15:40:31
category: rails
---

在分享arel使用的[后续补完](/intro-of-orderquery-class)中，我设计了一个名为 OrderQuery 的类供订单类做为构建Sql的底层接口来调用，一定程度上简化了复杂Sql的构建。这东西看起来很不错，不过这东西存在几个问题：

1. 每次使用，都需要在 OrderQuery 上进行调用，也就是说我们无形中多实例化了一个类，而且使用起来也不怎么优雅
2. 这个类完全是针对于 Order 类去构建的，里面一些有用的方法实际上是可以抽象公用的

要同时解决这两个问题，2B青年的做法是：直接将相关代码粘贴到所需要的类里面去（什么？你说你有50个类需要使用这个？……）。身为文艺青年的我们当然不会怎么干了。我们的解决之道是，使用ruby解决多重继承的方案-Mixin，将代码抽象为module后，混入对应的类中。

传统的混入module的代码是类似下面这样的：

```ruby
module FeatureEx
  module InstanceMethods
  # 声明一个实例方法
    def new_instance_method
      'this is a new instance method!'
    end
  end

  module ClassMethods
  # 声明一个类方法
    def new_class_method
      'this is a new class method!'
    end
  end

  def self.included(base)
    base.send(:include, InstanceMethods)
    base.extend ClassMethods
  end
end

class Feature
  include FeatureEx
end

feature = Feature.new
feature.new_class_method # => "this is a new class method!"
feature.new_instance_method # => "this is a new instance method!"

```

通过上面示例可看到，FeatureEx 模块已经被成功混入 Feature 类。让我们来稍微研究下 FeatureEx 是怎么被混入 Feature 的。

在上面的示例代码中，当 Feature 类被初始化时，会执行 include，加载所需引用的类/模块。而 FeatureEx 中的 included 是一个 Module 类的回调方法，当其被其他类 include 时，included 回调方法中的内容会自动执行。

included 有一个参数 base，base 用以指代调用了 include 方法的那个类，在上面的例子中，即是指 Feature。通过 base，我们可以很方便的使用 send 和 extend 两个方法将之前所定义的类方法以及实例方法混入 base 所指代的类中。

这一经典的 module 样板被广大群众们广泛使用，但实际上这一混入 module 的方式也存在几个问题：

 - 首先，如果被混入的 module 存在继承关系，则需要依次将全部关联 module 都手动 include 进来，否则会发生问题。请看下面一段代码：

```ruby
module FeatureEx
  def self.included(base)
    # 此处我们期望被 include 之后，调用 Feature 类的 greeting 属性
    base.greeting = "welcome, buddy!"
  end
end

module FeatureEx2
  include FeatureEx
end

class Feature
  # 我们向类内部添加一个名为 greeting 的属性用于测试
  class << self
    attr_accessor :greeting
  end
  # 将 FeatureEx2 混入进来
  include FeatureEx2
end

# 执行这个会发生什么？
Feature.greeting # => ?

```

按照惯性思维，因为 FeatureEx2 包含了 FeatureEx， 当将 FeatureEx2 混入 Feature 后，Feature 应该完好得继承了这两个 module 的特性，所以理论上，访问 greeting 应该会得到"welcome, buddy!"的结果。但实际上我们得到了如下错误：

```ruby
  NoMethodError: undefined method `greeting=' for FeatureEx2:Module'`
```

为什么会这样？因为如最开始所说，included 方法是个回调方法，它会在 module 自身被混入时立即执行，而 FeatureEx 是在 FeatureEx2 中被混入的，根据上下文关系，FeatureEx 所调用的参数 base 就变成了 FeatureEx2！而 FeatureEx2 中当然不可能有 greeting 方法，所以我们吃到了上面这样的错误。

解决之道就是乖乖的 included 进全部模块。

```ruby
class Feature
  class << self
    attr_accessor :greeting
  end

  # 除此之外还要删掉 FeatureEx2 中的 include
  include FeatureEx
  include FeatureEx2
end
```

 - 其次，如果没有特殊需求，每当新增加一个 module，我们都要写上下面这样千篇一律的包含代码，而这违背了DRY原则。

```ruby
def self.included(base)
  base.extend ClassMethods
  base.class_eval do
    # 此处包含实例方法
  end
end
```

如果说问题2还不算大问题，那么问题1就很严重了。这在编写代码时会带给你非常糟糕的体验，因为你要不断向上追溯，到底你需要引用的 module 包含了那些其他 module。想想为什么在linux下编译安装一些程序非常痛苦吧，因为那些乱七八糟的依赖关系让人抓狂，这也是为啥那些包管理器大行其道的原因所在。

实际上，在 rails 中实际上早就有了解决方法，那就是使用 ActiveSupport::Concern 这个模块。来看看我们刚才的示例使用 Concern 改写一番后是什么样子。

```ruby

module FeatureEx
  extend ActiveSupport::Concern

  included do
    # 此处不再需要使用 base，直接使用 self 关键字引用上下文即可
    self.greeting = "welcome, buddy!"
  end
end

module FeatureEx2
  extend ActiveSupport::Concern

  include FeatureEx
end

class Feature
  class << self
    attr_accessor :greeting
  end

  # 只需混入 FeatureEx2
  include FeatureEx2
end

```

看起来很神奇，不但简化了依赖关系，而且只用了一个 included do 就把类方法和实例方法的混入都解决了，那么 rails 具体是怎么做到的呢？让我们深入 Concern 模块的源代码一窥究竟。

```ruby
module ActiveSupport
  module Concern

    class MultipleIncludedBlocks < StandardError #:nodoc:
      def initialize
        super "Cannot define multiple 'included' blocks for a Concern"
      end
    end

    def self.extended(base) #:nodoc:
      base.instance_variable_set(:@_dependencies, [])
    end

    def append_features(base)
      if base.instance_variable_defined?(:@_dependencies)
        base.instance_variable_get(:@_dependencies) << self
        return false
      else
        return false if base < self
        @_dependencies.each { |dep| base.send(:include, dep) }
        super
        base.extend const_get(:ClassMethods) if const_defined?(:ClassMethods)
        base.class_eval(&@_included_block) if instance_variable_defined?(:@_included_block)
      end
    end

    def included(base = nil, &block)
      if base.nil?
        raise MultipleIncludedBlocks if instance_variable_defined?(:@_included_block)
        @_included_block = block
      else
        super
      end
    end

    def class_methods(&class_methods_module_definition)
      mod = const_defined?(:ClassMethods) ?
        const_get(:ClassMethods) :
        const_set(:ClassMethods, Module.new)
      mod.module_eval(&class_methods_module_definition)
    end
  end
end
```

一共不到40行代码，但是里面充斥着各种元编程的内容，所以我们按方法逐一进行分析。

```ruby
  def self.extended(base) #:nodoc:
    base.instance_variable_set(:@_dependencies, [])
  end
```

extended 和 included 一样，也是一个回调方法，不同之处在于它是在被其他类/模块 extend 时才会触发（见ruby的[相关文档](http://ruby-doc.org/core-2.1.3/Module.html#method-i-extended)）。在这里，Concern 模块调用该方法，在自身被 extend 时，将一个名为 _dependencies 的实例变量注入混入了 Concern 模块的 base 所指向的类内。

```ruby
  def append_features(base)
    if base.instance_variable_defined?(:@_dependencies)
      base.instance_variable_get(:@_dependencies) << self
      return false
    else
      return false if base < self
      @_dependencies.each { |dep| base.send(:include, dep) }
      super
      base.extend const_get(:ClassMethods) if const_defined?(:ClassMethods)
      base.class_eval(&@_included_block) if instance_variable_defined?(:@_included_block)
    end
  end
```

append\_features 是一个很容易和 included 混淆的回调方法，两者都是在当前 module 被混入其他类/模块时发生，不同之处可见[这篇文章]()的测试，再次不过多赘述。在上面代码中，Concern 模块覆盖了这一方法，检测执行了混入操作的 base 类中是否存在 \_dependencies 这一实例变量，如果存在，则将当前被混入的模块加入 @\_dependencies 变量中；如果不存在，则说明该类/模块没有 extend Concern 模块，也就是该类为最终进行混入的类，那么如果确定当前模块不是 base 类的子类，则循环将当前模块 @\_dependencies 中存储的相关模块依次混入 base 类，之后调用 Module 模块默认的 append\_features 方法，然后将定义的相应类方法和实例方法混入 base。

```ruby
  def included(base = nil, &block)
    if base.nil?
      raise MultipleIncludedBlocks if instance_variable_defined?(:@_included_block)
      @_included_block = block
    else
      super
    end
  end
```

该方法会检测 @\_included\_block 这一实例变量是否存在，如果存在，则说明 module 内至少存在1个以上的 included 块，则抛出异常；否则就将块的引用保存在实例变量 \_included\_block 中。如果不存在 base 这一参数，则调用默认方法进行处理。

```ruby
  def class_methods(&class_methods_module_definition)
    mod = const_defined?(:ClassMethods) ?
      const_get(:ClassMethods) :
      const_set(:ClassMethods, Module.new)
    mod.module_eval(&class_methods_module_definition)
  end
```

rails4中新增加的语法糖，让你能够将

```ruby
module ClassMethods
  #...
end
```

中定义的类方法改成如下这种形式进行定义：

```ruby
# 我没看出这东西有啥存在的价值，大概是为了和 included do 统一风格吧……
class_methods do
  #...
end
```

虽然源码分析完了，但是很多人看完了估计还是会很晕。让我们结合前面 Feature 的例子，以流程图的形式捋顺下这一复杂的过程（yuml不支持中文什么的我才不会说呢OTL）。

![](http://yuml.me/64514ab9)

1. 我们触发了 Feature 类的加载，Feature 类执行 include 方法，载入 FeatureEx2
2. FeatureEx2 混入Concern模块，导致自己的 append\_features 和 included 两个方法被覆盖，同时，extended 回调被触发，@\_dependencies 被建立。回调完成后，FeatureEx2 执行下一行，载入 FeatureEx 模块。
3. FeatureEx 前几步的流程和 FeatureEx2 被载入时一样，完成同样工作后，FeatureEx 会执行被 Concern模块覆盖掉的 append\_features 回调。 唯一的不同是，因为上下文转换，当前的 base 变为了 FeatureEx2，而这正是关键所在。因为 FeatureEx2 中已经存在 @\_dependencies，所以 FeatureEx 将其自身加入到 @\_dependencies 后便结束了回调方法，程序又条回到了 FeatureEx2 模块中。
4. 因为没有其他代码可以执行了，所以 FeatureEx2 也不情愿得调用自己的 append\_features 来结束自己的使命。而因为此时 base 又变为了 Feature 类，所以检测 @\_dependencies 是否存在的条件语句转到 else 分支，在循环将 @\_dependencies 都加载进 Feature 类之后，再将自己混入 Feature，整个过程结束。

## 结语

在 rails4 中，DHH大力推行 Concern 的使用，以至于 Concern 已经被剥离到独立的目录存放，甚至扩展到 Controller 中使用，这一模式确实进一步降低了代码的耦合度，提高了代码的可维护性。
