---
layout: post
title: "记to_xml中的一个坑"
date: 2014-10-29 15:03:31
category: ruby
---

这几天在做支付宝的手机版即时到帐接口。因为手机版接口要求请求业务参数（req\_data）为xml格式，所以很自然的就想到了先把参数放在哈希中，然后使用 to\_xml 搞定。看着php版demo里那一坨直接拼接的 req\_data，在沉醉于ruby代码之优雅中，完成了授权接口，开始调试。

```ruby
  parameters[:req_data] = { # params here }.to_xml(skip_instruct: true, skip_types: true, root: 'direct_trade_create_req', indent: 0)
```

结果很快不幸就降临了：发送数据后，响应回来的是0004错误（req_data格式不正确）。由于文档里写的也很模糊，我一度以为req\_data中的参数也需要正序排列，所以反复折腾了半天排序也没能解决问题……

浪费了大量时间后，问题依旧，无奈搭了个php环境，把官方demo生成的请求链接和自己生成的链接仔细比对了下，发现了一个很囧的现象：参数里有的下划线都被莫名其妙转化成了连字符OTL

翻了下rails的文档，发现 to_xml 有个 :dasherize 参数，默认值为 true。当其值为 true时，会自动将属性名中所有的空格和下划线转化为连字符。

于是添加了该参数，并将其置为 false，问题解决。不过最终也没想明白为什么 rails要做这样的设置，因为在xml中，下划线也可以合法用于属性名……
