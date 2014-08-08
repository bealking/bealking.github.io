---
layout: post
title: "rails中的序列化功能以及应用"
date: 2014-08-01 11:40:41
category: ruby
---

重构系列进入了第三篇，今天要讲的主题是：rails中的序列化。开篇当然还是以我们在浣熊市的伟大超市的故事切入。

## 喜闻乐见的故事（还有没有完？）

在我们开发出了伟大的表格导入功能后，那些卑鄙无耻的竞争对手被我们打的落花流水，我们的超市也继续蓬勃发展。不过这些小人不会心甘情愿接受自己被击败的事实，于是又以五花八门的促销活动向我们发起了新一轮挑战：花脸超市的购物返现活动风生水起、我二妈超市的买赠活动如火如荼、霾得聋的加价购也搞得轰轰烈烈。于是正当我们享受着午餐美味的排骨炖苍蝇时，咆哮的老板不适时地将我们拖进了办公室，限令我们一周之内将对手的全部促销活动都开发出来，并且还要比他们的功能更强大。

脑子里一团乱麻的我们开始规划起了我们的促销活动功能设计：

1. 这些活动本质上都是促销手段，应该放置在一张表里（一个父类），必要时候使用单表继承去进行扩展。这样在活动管理上也会方便很多，也避免了重复增加控制器和模型。
2. 这些活动可能需要大量的字段来记录活动的相关属性规则。比如：返现活动需要名为refund_money的字段记录返现金额；赠品功能需要一个名为gifts的字段记录赠品，或者是新的子表来记录；加价购需要一个名为products的字段，甚至是一个子表来记录参与加价购的商品……

想到这里我们已经崩溃了，因为我们仅仅为了满足3个活动的最基本需求就可能需要增加2张表&3个字段或者是3个字段以及解析串化数据的方法了。如果再想想我们的活动可能还有类似于每人限购几件（card_limit）、每个订单最低需要达到多少钱（order_limit）这样的细节需求，那么我们的表的字段可能要高达几十个。这样的表管理和维护起来都是很大的麻烦，更别说操作这么复杂的表的代码了。

## 使用ActiveRecord::Store简化表的复杂度

实际上在rails中已经有这样应用场景的解决方案了，那就是今天要讲的主题：ActiveRecord::Store。

我们知道，ActiveRecord提供了内置的[序列化方法](http://api.rubyonrails.org/classes/ActiveRecord/AttributeMethods/Serialization/ClassMethods.html#method-i-serialize)"serialize"用以将串化后的数据存入文本字段中。但是这一功能过于单薄，我们将串化数据从库中取出后，还需要手动进行反序列化，将其转化为hash后才能进行操作。

而ActiveRecord::Store很好的解决了这一问题。Store类又在serialize的基础上进行了二次包裹（可看到被Store包裹的实例变量其实还是Serialization对象），使得我们可以在从库中取出数据后，不需要任何额外地处理就能直接访问到这些数据，而且就好像访问该对象的自有属性一样。

通过使用Store，我们可以构造一个类似下面的促销活动类。

```ruby
class Promot < ActiveRecord::Base
  store :settings, accessors: [:gifts, :products, :refund_money, :order_limit, :card_limit], coder: JSON
end
```

在这个类中，我们声明了一个用于储存串化数据的数据库字段，这个字段名为:settings。他实际上是一个存储器，用于存储我们所声明的一系列属性。在rails4中，还可以使用coder参数指定存储时使用的格式（JSON或YAML），不过在rails3中，只支持存储为YAML格式。

完成声明后，我们既可以像使用该类自有的属性一般进行创建、赋值等操作。

```ruby
  promot = Promot.new(gifts: [172,48,59], order_limit: 30, card_limit: 2)
  promot.gifts #=> [172, 48, 59]
  # 修改值
  promot.gifts = [56,78] #=> [56,78]

  # 也可以用哈希形式访问存储器本身来获取值
  promot.settings[:gifts] #=> [172, 48, 59]
  # 也可以用字符串的key
  promot.settings['gifts'] #=> [172, 48, 59]

  # 存储时会自动串化保存
  promot.save #=> INSERT INTO `promots` (`created_at`, `settings`, `updated_at`) VALUES ('2014-08-08 06:21:10', '{\"gifts\":[172,48,59],\"order_limit\":30,\"card_limit\":2}', '2014-08-08 06:21:10')

  # 读取时会自动进行反序列化
  Promot.last #=> => #<Promot id: 1, settings: {"gifts"=>[172, 48, 59], "order_limit"=>30, "card_limit"=>2}, created_at: "2014-08-08 06:21:10", updated_at: "2014-08-08 06:21:10">
```

rails4中还可以通过"stored_attributes"方法来获取存储器中所存储的accessor名称。这在进行一些判断处理的时候非常有用处。

```ruby
  Promot.stored_attributes[:settings] #=> [:gifts, :products, :refund_money, :order_limit, :card_limit]
```

此外还可以覆盖Stroe对accessor默认的取值和赋值处理，来实现自己所需要的逻辑（rails4）。在类Promot中定义以下两个方法。

```ruby
  # 存储order_limit时，将其值强制取整
  def order_limit=(value)
    super(value.to_i)
  end

  # 读取order_limit时，将其值翻倍
  def order_limit
    super.to_i * 2
  end
```

试验一下结果：

```ruby
  promot.order_limit = 'a'
  promot.order_limit #=> 0

  promot.order_limit = '3'
  promot.order_limit #=> 6
```

## 结语

### ActiveRecord::Store的优点

1. 可以大大降低一些场景应用下的数据库设计复杂度，同时提供超高扩展性，对于一些在高速迭代，不断变化中的需求非常强的支援
2. 提高代码的可维护性以及可读性
3. 提高数据库空间使用率，减少冗余数据

### ActiveRecord::Store的缺点

因为属性被放在寄存器内串化，所以将不能很好得支持sql查询，丧失大量sql特性。
