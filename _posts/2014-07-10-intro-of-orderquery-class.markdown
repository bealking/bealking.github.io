---
layout: post
title: "重构复杂查询（续）——OrderQuery类的使用说明"
date: 2014-06-16 17:40:41
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


