---
layout: post
title: "重构复杂查询（续）——OrderQuery类的使用说明"
date: 2014-06-20 14:40:41
category: rails
---

上一次分享中，讲到Arel重构复杂查询时，拿订单查询类中的查询封装举了例子。

``` ruby
class OrderQuery

  SALE_ORDER = Order.enum_sale_kinds.values_at(:subs, :spot, :groupon)
  SALED_STATUSES = Order.enum_statuses.values_at(:pending, :printing, :arranging, :shipping, :finished, :ready)
  REFUNDED_STATUS = Order.enum_statuses[:refunded]

  def initialize()
    @order = Order.arel_table
  end

  # 财务视为已售出状态的订单
  def saled(include_refunded=true)
    _statuses = include_refunded ? SALED_STATUSES << REFUNDED_STATUS : SALED_STATUSES
    @order[:sale_kind].in(SALE_ORDER).and(@order[:status].in(_statuses))
  end

  # 财务视为已退款状态的订单
  def refunded
    @order[:sale_kind].in(SALE_ORDER).and(@order[:status].eq(REFUNDED_STATUS))
  end

  # 由指定终端销售的订单
  # @param depots 需要查询的终端，可为数组或整数
  def saled_by(depots)
    return '' if depots.blank?
    depots.is_a?(Array) ? @order[:source_id].in(depots).or(@order[:proxy_id].in(depots)) : @order[:source_id].eq(depots).or(@order[:proxy_id].eq(depots))
  end

  #... other query methods ...
end
```


当时所举的使用订单查询类替换通常查询的例子：

通常查询

``` ruby
  saled_orders = Order.where('proxy_id = ? or source_id = ?', params[:id], params[:id])
                      .where('created_at between ? and ?', params[:started], params[:ended])
                      .where(depot_id: params[:depot_id])
```
使用了订单查询类之后……

``` ruby
  q = OrderQuery.new
  saled_orders = Order.where(q.saled)
                      .where(q.time_range(:created_at, params[:started], params[:ended]))
                      .where(q.saled_by(params[:depot_id]))
```

不过，考虑到这个查询实际上是可以用ransack进行处理的：

``` ruby
  saled_orders = Order.ransack({
      proxy_id_or_source_id_eq: params[:id],
      created_at_gt: params[:started]
      created_at_lt: params[:ended]
      depot_id_in: params[:depot_id]
  }).result
```

所以Arel的优势体现的不是很明显。直到上周，伟大的财务要求进行全部已审核完毕的退货单的统计，我感觉机会来了XD。

先简单说下退货单。仓库的退货单分为两种：

1. 因临期等原因需要返还供应商进行调换的，此类视为供应商经由采购向仓库要货，财务扮演监察者的角色
2. 部分商品因破损或者丢失而无法平仓的，仓库提交退货申请后，经由财务进行审核通过后，会变为盘亏订单

如果用arel直接构造条件，查询退货单，那么可能会诞生出类似下面的方法：

``` ruby
  def wareh_returned
                      # 条件2
    @order[:sale_kind].eq(Order.enum_sale_kinds[:deficit]).and(@order[:status].eq(Order.enum_statuses[:finished]))
                      # 条件1
                      .or(@order[:sale_kind].eq(Order.enum_sale_kinds[:purchase_refunded]).and(@order[:status].eq(Order.enum_statuses[:returned])))
  end
```

写完之后，如果仔细观察这一新增的方法，你会发现，他的结构和之前的方法很像，都是查询"status"和"sale_kind"在范围内的某些订单，唯一不同是将两种单子用or进行了拼接。如果之后还有类似的查询接口要暴露出去，那么又会增加类似方法，稍微复杂一些的就会生成类似于我们新写的这个和saled_by方法中的代码。这违反了DRY原则，也增加了代码维护的难度（试想你在一堆括号中判断逻辑关系的情景……）

解决这一尴尬问题的途径就是将类进行DSL化重构。元编程配合arel的强大，可以产生出你自己的一套Sql DSL。

重构后的OrderQuery类似下面这样：

```ruby

class OrderQuery

  SALE_ORDER = [:subs, :spot, :groupon]
  SALED_STATUSES = [:pending, :printing, :arranging, :shipping, :finished, :ready]
  REFUNDED_STATUS = :refunded

  def initialize()
    @order = Order.arel_table
  end

  # 财务视为已售出状态的订单
  def saled(include_refunded=true)
    _statuses = include_refunded ? SALED_STATUSES << REFUNDED_STATUS : SALED_STATUSES
    build_condition(:and, {sale_kind: SALE_ORDER, status: _statuses})
  end

  # 财务视为已退款状态的订单
  def refunded
    build_condition(:and, {sale_kind: SALE_ORDER, status: [REFUNDED_STATUS]})
  end

  # 财务视为已审核状态的仓库退货单
  # 仓库的退货单分为两种：
  #   1、因临期等原因需要返还供应商进行调换的，此类视为供应商经由采购向仓库要货，财务扮演监察者的角色
  #   2、部分商品因破损或者丢失而无法平仓的，仓库提交退货申请后，经由财务进行审核通过后，会变为盘亏订单
  def wareh_returned
    build_condition(:and, {sale_kind: [:deficit], status: [:finished]}).or(build_condition(:and, {sale_kind: [:purchase_refunded], status: [:returned]}))
  end

  # 由指定终端销售的订单
  # @param depots 需要查询的终端，可为数组或整数
  def saled_by(depots)
    return '' if depots.blank?
    depots = Array(depots)
    build_condition(:or, {source_id: depots, proxy_id: depots})
  end

  # 某date型字段一定时间范围内的订单
  # @param [Symblo] column 要查询的字段名
  # @param [String] started 要查询的起始时间，支持多种格式，如：'2014-05-26' 或 '2014/05/26'
  # @param [String] ended 要查询的结束时间，例如：'2014-05-27'
  # @param [Boolean] format_ended 是否自动格式化结束时间到当日结束
  # @return [ActiveRecord::Relation]
  # @example 查询退款时间为14年5月24日到14年5月26日的订单
  #   query = OrderQuery.new
  #   Order.where(query.time_range(:refunded_at, '2014-05-24', '2014-05-26')  #=> [#<Order>]
  def time_range(column, started, ended=nil, format_ended: true)
    _started = started.blank? ? Time.now.beginning_of_day : Time.parse(started)
    _ended = if ended.blank?
               _started.end_of_day
             else
               format_ended ? Time.parse(ended).end_of_day : Time.parse(ended)
             end
    @order[column].gteq(_started).and(@order[column].lteq(_ended))
  end

private
  # 根据传入的条件哈希组构建arel节点并返回
  # @param [Symblo] operator 操作符，or/and
  # @param [Hash] args 要查询的字段为索引的符号值集合，详见下面示例
  # @return [ActiveRecord::Relation]
  # @todo 更强的参数形式支持
  # @example 查询门店id为425的已取消或者已退款状态的前10条预订单
  #   query = OrderQuery.new
  #   Order.where(order.build_condition(:and, {sale_kind:[:subs],status:[:cancel,:refunded],proxy_id:[425]})).limit(10)
  #   #=> SELECT `orders`.* FROM `orders` WHERE (`orders`.`sale_kind` IN (0) AND `orders`.`status` IN (6, 7) AND `orders`.`proxy_id` IN (425)) LIMIT 10
  def build_condition(operator, *args)
    args.extract_options!.inject(nil) { |result, (key, val)|
      # 将enum值转化为对应数值
      val = symtoval(key, *val) if Order.respond_to?("enum_#{key.to_s}".pluralize.to_sym)
      if result.nil?
        result = val.length == 1 ? @order[key].eq(val.take(1)) : @order[key].in(val)
      else
        result = result.send(operator, val.length == 1 ? @order[key].eq(val.take(1)) : @order[key].in(val) )
      end
    }

  end

  # 将订单类某字段的符号值转为对应的数值
  # @param [Symblo] column 要查询的字段名
  # @param [Symblo] symblo_array 所需查询字段的enum符号数组
  # @return [Array] 对应符号的数值组
  # @example 查询status字段，:finished和:pending状态对应的值
  #   symtoval(:status, :finished, :pending) #=> [5,0]
  def symtoval(column, *symblos)
    Order.send("enum_#{column.to_s.pluralize}").values_at(*symblos).uniq.compact
  end
end

```

通过增加build_condition这一方法，很好地达成了DRY原则。也为之后可能出现的类似方法提供了很好的抽象支持，如果暴露为外部接口，则能大大简化Sql查询。
