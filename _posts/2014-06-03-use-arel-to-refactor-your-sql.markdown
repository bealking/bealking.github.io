---
layout: post
title: "如何重构复杂SQL查询"
date: 2014-06-16 17:40:41
---

# 一个故事……
## 美好的开始

设想我们是一家小超市的程序员，超市每日的流水都保存在一张订单表中，财务要求看到所有的订单数据以便进行每日的应收账款对账工作。于是初始的查询是下面这个样子的：

{% highlight ruby %}
orders = Order.all #=> 轻而易举得就得到了全部订单，很好，很强大！：D
{% endhighlight %}

超市的事业蓬勃发展，一年内，我们的雇员从之前的1人增加到100人，几家分店崛地而起，于是财务告诉我们，现在他要分店分人进行订单统计，解决这个问题似乎也不困难，因为我们未卜先知的保存了订单所在的门店号和雇员工号，于是我们的查询代码稍加改动就打发了财务：

{% highlight ruby %}
orders = Order.where(depot_id: params[:depot_id], employee_id: params[:employee_id]).all #=> everything is under my control!
{% endhighlight %}

于是我们又逍遥了数日。不过从不满足的财务又提出了新的需求：按日统计，于是我们的代码变成了这样：

{% highlight ruby %}
orders = Order.where(depot_id: params[:depot_id], employee_id: params[:employee_id])
              .where(created_at: params[:created_at])
              .all #=> 事情看起来开始有点不美了……
{% endhighlight %}

## 遭遇的问题

我们传了一堆参数来构造查询条件，但是这些参数并不是完全必须的，如果其中有任意一项为空，那么我们就可能吃到一个**ActiveRecord::StatementInvalid**。于是我们的代码可能会变成这个样子：

{% highlight ruby %}
orders = Order.scoped
orders = orders.where(depot_id: params[:depot_id]) if params[:depot_id].present?
orders = orders.where(employee_id: params[:employee_id]) if params[:employee_id].present?
# 更多更恶心的条件拼接...
{% endhighlight %}

如此下去，我们的代码将变的无限臃肿。不过好在不可能只有我们自己碰到了此类问题，于是我们可以站在[巨人的肩膀](https://github.com/activerecord-hackery/ransack)上。用ransack重构代码后，我们获得了暂时的解脱XD

{% highlight ruby %}
orders = Order.search(params[:q]).all
# params[:q] => {depot_id_eq: nil, employee_id_eq: nil, created_at_eq: nil}
{% endhighlight %}

## 但是……

商品不是完美的，所以总有顾客买了东西后来进行退货。于是我们忙碌的店员在销售之余还要接受顾客退回来的东西，并将相应的款项退还给顾客。
商品不完美，我们使用的Gem也不是完美的。看似美好的ransack只支持And，于是在财务告诉我们，除了要统计每日销售的订单外，还要统计一下顾客的退单以便保证每天店员的营业款足够准确的时候，我们崩溃了。因为我们优雅的代码可能会变成这样：

{% highlight ruby %}
orders = Order.search(params[:q]) # params[:q] => {depot_id_eq: nil, employee_id_eq: nil}
orders = Order.where('created_at = ? or refunded_at = ?', params[:created_at], params[:created_at]) if params[:created_at].present?
orders.all
{% endhighlight %}

## What the fuck!

之后财务还可能会提出关联其他表的需求，于是你会发现where中的条件因为没写表名而存在冲突……

![](../images/wtf.png)

# 救世主的伟大登场
## Arel是什么？

Arel是一个Ruby的Gem，它是个SQL AST管理器。
AST 是抽象语法树（Abstract syntax tree）的缩写。

Arel一开始由[Bryan Helmkamp](https://github.com/brynary/arel)维护开发，由于其强大的功能，而被Rails开发小组[forked](https://github.com/rails/arel)，吸收进了其框架内并注入AR，而成为Rails源代码的一部分。

## 用Arel拯救被毁掉的查询

{% highlight ruby %}
def table
  Order.arel_table
end

def employee_eq(employee_id)
  table[:employee_id].eq(employee_id)
end

def depot_eq(depot_id)
  table[:depot_id].eq(depot_id)
end

def created_in(time)
  table[:created_at].eq(time)
end

def refunded_in(time)
  table[:refunded_at].eq(time)
end

orders = Order.where(employee_eq(params[:employee_id]))
              .where(depot_eq(params[:depot_id]))
              .where(created_in(params[:created_at]).or(refunded_in(params[:created_at])))
              .all
{% endhighlight %}

## 抽象查询的好处

- 可读性
- 重用性
- 高可维护性
- 稳定性

# 路在脚下，更深度的挖掘Arel
