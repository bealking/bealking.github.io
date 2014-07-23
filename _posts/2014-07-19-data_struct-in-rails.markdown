---
layout: post
title: "浅谈在rails中使用结构体进行重构"
date: 2014-07-19 11:40:41
category: rails
---

在谈重构的[第一篇文章](/use-arel-to-refactor-your-sql)中以一个超市的例子对用arel重构查询进行了简单的说明，因为重构后的例子不是太有说服力，所以后面又[追文一篇](/intro-of-orderquery-class)进行了补充说明。今次仍然是继续这个故事来引出今天要说的主题。

## 喜闻乐见的故事（续）

在使用arel进行了重构后，我们的连锁超市如火如荼地扩展开来，已经制霸了浣熊市多条主要街道，俨然成为了当地零售百货业的龙头企业（伪）。我们的生意愈发红火的同时，同行嫉妒的眼光也接踵而至，无耻地和我们搞起了价格战，我们卖3块的东西他们就降价到2块8。

拜价格战所赐，我们可怜的产品编辑通宵忙碌于更改各种产品的售价、成本，以至于连续数日没有睡觉而精神崩溃提出了辞职！于是老板拍着桌子咆哮着限令我们2天之内搞出一个可以通过excel导入来自动更新产品信息的系统，这样他就可以花民工价雇佣几个只懂得使用土鳖excel的菜鸟大学生来完成这些操作了。我们屁滚尿流地从老板办公室滚出来后，扔下手中刚跑了一半的酷跑，开始了新功能开发。

## 上头定制的数据表格的样品

<table style="border-collapse: collapse" bordercolor="#ccc" cellspacing="0" cellpadding="4" width="100%" align="center" border="1
">
  <thead>
    <tr class="header">
      <th align="left">商品ID</th>
      <th align="left">商品名称</th>
      <th align="left">售价</th>
      <th align="left">成本</th>
    </tr>
  </thead>
  <tbody>
    <tr class="odd">
      <td align="left">100639</td>
      <td align="left">百事可乐可乐型汽水2.5L</td>
      <td align="left">8.30</td>
      <td align="left">7</td>
    </tr>
    <tr class="even">
      <td align="left">100927</td>
      <td align="left">冰花糖水白桃罐头300g</td>
      <td align="left">5.10</td>
      <td align="left">4.5</td>
    </tr>
    <tr class="odd">
      <td align="left">100929</td>
      <td align="left">冰花糖水橘子罐头300g</td>
      <td align="left">5.30</td>
      <td align="left">4.5</td>
    </tr>
  </tbody>
</table>


## 感觉貌似良好的第一版

```ruby
def update_by_csv_date(file)
  #这里我们假设文件上传到了项目根目录的public下
  _data = CSV.read( "#{Rails.root}/public/items.csv", encoding: "GBK:UTF-8", skip_lines: /^\p{Han}+/)
  #=> 我们会得到类似于下面这样的一个数组
  #=> [["100639", "百事可乐可乐型汽水2.5L", "8.30 ", "7"], ["100927", "冰花糖水白桃罐头300g", "5.10 ", "4.5"], ["100929", "冰花糖水橘子罐头300g", "5.30 ", "4.5"]]
  _data.each{|line|
    Product.find(line[0]).update_attributes(name: line[1], price: line[2], inbound_price: line[3])
  }  #开发环境测试需要设置身份: Employee.current = Employee.find(2)

```

代码很不错，可以正常工作，完成了老板的要求，为老板节省工资立下了汗马功劳，但是……
我们给自己埋下了一些可维护性上面的隐患：我们利用ruby内置的Csv类，直接将csv表格以数组的形式读入并开始操作，这样就使得我们的代码内存在大量类似于"line[n]"这样带着数组键值的变量，使我们代码的可读性变得十分低下。以上例子只是个简单的示范，如果还有校验表格数据合法性、处理数据格式等复杂的逻辑的话，我们的代码内会充斥着这样的变量，导致后期维护和阅读起来会非常困难。

于是，我们对代码进行了简单的改造：

## 具备国际顶尖水平的第二版

```ruby
class ImportCsv

  # 读入csv文件
  def initialize(file)
    @data = CSV.read("#{Rails.root}#{file}", encoding: "GBK:UTF-8", skip_lines: /^\p{Han}+/)
  end

  # 转化为哈希组供后续操作
  def to_hash
    @data.inject({}) {|hash, line|
      hash[line[0]] = {
        name: line[1],
        price: line[2],
        inbound_price: line[3],
      } unless hash.has_key?(line[0])
      hash
    }
  end
end

```

我们声明了一个名为"ImportCsv"的类进行csv导入，该类中有一个名为"to_hash"的方法，可以将导入的数组转为哈希形式，供后续处理.

```ruby

def update_by_csv_date(file)
  _hash = ImportCsv.new("public/test.csv").to_hash
  #=> {"100639"=>{:name=>"百事可乐可乐型汽水2.5L", :price=>"8.30 ", :inbound_price=>"7"}, "100927"=>{:name=>"冰花糖水白桃罐头300g", :price=>"5.10 ", :inbound_price=>"4.5"}, "100929"=>{:name=>"冰花糖水橘子罐头300g", :price=>"5.30 ", :inbound_price=>"4.5"}}
  _hash.each do |key, val|
    Product.find(key.to_i).update_attributes(val)
  end
end

```

经过这番修改后，我们的代码看起来比第一版好多了，将导入逻辑抽象成了单独的类，更新模型的代码也更加的干净和可读。但是实际上ImportCsv类里的代码还是存在一些问题：

1. line[n]的问题还是没能解决，这样实际上是“金玉其外败絮其中”的解决办法
2. 我们通过注入生成一个哈希组来格式化生成所需要操作的字段组。一旦需要更新的字段很多时，我们需要注入的字段也随之增长，并变成类似下面这样：

```ruby

  hash[line[0]] = {
    name: line[1],
    price: line[2],
    inbound_price: line[3],
    quantity: line[4],
    barcode: line[5],
    # more columns...
  } unless hash.has_key?(line[0])``ruby

```

![](../images/gomad.jpg)


## 使用结构体构造宇宙无敌版

有没有更好的解决办法呢？实际上是有的，那也就是我们今天要讲的东西：结构体。结构体是很多语言中都有的东西，比如C。结构体可以简单快速的存储一组数据而不用声明一个类。ruby中的结构体分为两种，一种是核心库中用C编写的[Struct](http://www.ruby-doc.org/core-2.1.2/Struct.html)，另一个则是标准库中的[OpenStruct](http://www.ruby-doc.org/stdlib-2.1.2/libdoc/ostruct/rdoc/OpenStruct.html)。

Struct的使用：

```ruby
  CsvData = Struct.new(:name, :price, :inbound_price)
  data = CsvData.new('百事可乐可乐型汽水2.5L', 8.3, 7)
  data.name #=> "百事可乐可乐型汽水2.5L"
  # 可以直接转化为哈希或者数组
  data.to_h #=> {:name=>"百事可乐可乐型汽水2.5L", :price=>8.3, :inbound_price=>7}
  data.to_a #=> ["百事可乐可乐型汽水2.5L", 8.3, 7]
```
Struct由于使用C编写，所以性能很强劲，部分场景下的效能要比数组高很多。不过缺点也有：不能够动态向其中添加属性，没有序列化等的高级特性支持。而这些问题在OpenStruct中得到了解决。

OpenStruct的使用：

```ruby
  require 'ostruct'
  CsvData = OpenStruct.new
  CsvData.name = "百事可乐可乐型汽水2.5L"
  CsvData.inbound_price = 7
  CsvData.price = 8.3
  CsvData #=> #<OpenStruct name="百事可乐可乐型汽水2.5L", price=8.3, inbound_price=7>
  #可以直接对其进行序列化，变为哈希供存储或者操作
  CsvData.marshal_dump() #=> {:name=>"百事可乐可乐型汽水2.5L", :price=>8.3, :inbound_price=>7}
  #也可以直接反序列为OpenStruct对象
  CsvData.marshal_load({name:'可口可乐2.5L', price:9.3, inbound_price:7.5})
  CsvData.name #=> '可口可乐2.5L'
```

而通过使用Struct，我们上面的代码将会变得清晰而且简洁很多：

```ruby
class ImportCsv
  # 声明一个用于存放将要导入的表格中每行数据的结构体
  Line = Struct.new(:id, :name, :price, :inbound_price)

  # 读入csv文件
  def initialize(file)
    @data = CSV.read("#{Rails.root}#{file}", encoding: "GBK:UTF-8", skip_lines: /^\p{Han}+/)
  end

  # 转化为对象供后续操作
  def to_obj
    @data.collect{|line| Line.new(*line)}
  end
end

```

添加了to_obj方法后，我们的update_by_csv_date方法将可以修改成这样：

```ruby

def update_by_csv_date(file)
  _array = ImportCsv.new("public/test.csv").to_obj
  #=> [#<struct ImportCsv::Line id="100639", name="百事可乐可乐型汽水2.5L", price="8.30 ", inbound_price="7">, #<struct ImportCsv::Line id="100927", name="冰花糖水白桃罐头300g", price="5.10 ", inbound_price="4.5">, #<struct ImportCsv::Line id="100929", name="冰花糖水橘子罐头300g", price="5.30 ", inbound_price="4.5">]
  _array.each do |record|
    Product.find(record.id).update_attributes(record.to_h)
  end
end

```

怎么样，是不是清爽了很多呢？

## 结语

在特定的应用场景中使用正确的方法来处理，一些困扰我们的问题将会迎刃而解。ruby标准类中有很多类库值得我们去发掘，一味的重复发明轮子不但浪费自己的宝贵时间，也会给自己带来无穷尽的麻烦。

Struct和OpenStruct其实分别对应不同的应用场景，Struct很适合用来处理上面这种结构相对固定的场景，而OpenStruct由于具有更好的扩展性以及序列化特性，很适于去处理需求不断变化，功能迭代较快的场景，或者有序列化需求的场景，比如：购物车。
