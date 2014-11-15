---
layout: post
title: "在Rails中同时使用符号和字符串形式的key操作hash"
date: 2014-11-13 16:00:11
category: rails
---

我们可能有时会遇到这样的问题：当访问一个 hash 时，如果其中的元素使用了字符串形式的 key，而我们使用符号 key进行调用，就会无法取到值。

```ruby
  hash = { 'type' => 'mobile' }
  hash['type'] => "mobile"
  # 用符号key会返回 nil
  hash[:type] => nil
```

这种问题会出现在诸如API升级等场景中，比如转化 xml 为 hash 时。设想当你需要把一个 xml 串转化成 hash。

```ruby
  hash = Hash.from_xml("<errors><column>name</column><error>can't be blank</error></errors>")
  hash['errors'] #=> {"column"=>"name", "error"=>"can't be blank"}
  hash[:errors] #=> nil
```

当然，rails 给我们提供好了现成的 symbolize\_keys / deep\_symbolize\_keys 方法来把字符串形式的 key 转化为符号形式。

```ruby
  hash.deep_symbolize_keys! #=> {:errors=>{:column=>"name", :error=>"can't be blank"}}
```

这方案似乎不错，但是转换后字符串形式的 key 就无法使用了。设想如果你在升级一个类的接口方法，而这个方法返回一个以字符串为 key 的哈希，此时你想改为返回符号形式的 key，如果你使用了 symbolize\_keys 进行转化，那么所有涉及调用了该方法的地方都需要进行同步改动！

有没有鱼与熊掌可兼得的解决方案呢？答案当然是有！还记得我们之前讲到过的 [ActiveRecord::Store](/intro-of-activerecord-store/) 吗，包裹在 store 中的数据是同时支持两种形式的 key 来进行访问的。它是怎么做到的呢？我们分析下 Store 类的源代码，可以发现下面这么一段：

```ruby
   def dump(obj)
     @coder.dump self.class.as_indifferent_hash(obj)
   end
   def load(yaml)
     self.class.as_indifferent_hash(@coder.load(yaml || ''))
   end
   def self.as_indifferent_hash(obj)
     case obj
     when ActiveSupport::HashWithIndifferentAccess
       obj
     when Hash
       obj.with_indifferent_access
     else
       ActiveSupport::HashWithIndifferentAccess.new
     end
   end
```

不难发现，被 Store 序列/反序列化的数据都是通过 as\_indifferent\_hash 方法进行的，而 as\_indifferent\_hash 方法实际上只是对传入的对象类型进行了判断，返回包装过的数据。通过其中的分支判断我们可以发现，当对象为 ActiveSupport::HashWithIndifferentAccess 类型时，会不经任何处理直接返回。那么也就是说，ActiveSupport::HashWithIndifferentAccess 这个对象可以同时支持字符串和符号两种形式的 key。

通过查看[Rails的API文档](http://api.rubyonrails.org/classes/ActiveSupport/HashWithIndifferentAccess.html)，我们可以发现其实这货就是 Hash 类的一个子类。它将 Hash 类中很多会因为 key 类型不同而导致问题的方法都进行了重写。

设想这样一个场景，你有一个购物车的模型，其中可能以 Hash 形式保存商品数量，类似如下形式：

```ruby
  cart = {quantity: 3}
```

购物车接受一个更新数量的参数，对商品数量进行更新。不过当参数为字符串形式时，会发生问题：

```ruby
  # 假设我们的 params = {'quantity': 5}，那么我们会得到下面的结果……
  cart.update(params)  #=> {:quantity=>3, "quantity"=>5}
```

你需要手动将参数先 symbolize\_keys 才能避免遇到类似问题。HashWithIndifferentAccess 类重写了这些会产生类似问题的方法：

```ruby
  cart = ActiveSupport::HashWithIndifferentAccess.new(quantity: 3)
  cart.update('quantity' => 5)  #=> {"quantity"=>5}
  cart[:quantity]  #=> 5
```

这也正是Rails在 [actionpack 中所实现的 params 可以通过任意形式 key 来访问](https://github.com/rails/rails/blob/e595d91ac2c07371b441f8b04781e7c03ac44135/actionpack/lib/action_dispatch/http/parameters.rb)的奥秘所在。

```ruby
# /actionpack/lib/action_dispatch/http/parameters.rb 行 47
def normalize_encode_params(params)
  case params
  when Hash
    if params.has_key?(:tempfile)
      UploadedFile.new(params)
    else
      params.each_with_object({}) do |(key, val), new_hash|
        new_hash[key] = if val.is_a?(Array)
                          val.map! { |el| normalize_encode_params(el) }
                        else
                          normalize_encode_params(val)
                        end
      end.with_indifferent_access
    end
  else
    params
  end
end

# /actionpack/lib/action_dispatch/http/request.rb 行 298
# 重写了 rack 的 get 方法，以实现对两种形式 key 的支持
def GET
  @env["action_dispatch.request.query_parameters"] ||= Utils.deep_munge(normalize_encode_params(super || {}))
rescue Rack::Utils::ParameterTypeError, Rack::Utils::InvalidParameterError => e
  raise ActionController::BadRequest.new(:query, e)
end
```

之所以想贴出这段代码是因为，通过阅读这段代码，你会发现另外一个有用的方法：with\_indifferent\_access。

Rails将该方法以猴子补丁的形式打入了 Hash 类中，在这个方法的帮助下，你可以很轻易的将任意普通的 Hash 类的实例转化为 HashWithIndifferentAccess 对象。

```ruby
  hash = {quantity: 2}
  hash['quantity']  #=> nil
  hash = {quantity: 2}.with_indifferent_access
  hash['quantity']  #=> 2
```
