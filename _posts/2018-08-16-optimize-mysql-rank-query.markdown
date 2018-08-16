---
layout: post
title: "优化Mysql中的rank排序查询"
date: 2018-08-16 22:30:31
category: mysql
---

公司项目有一个签到功能，其中可以显示当前用户的连续签到次数在所有人中的排名。

因为之前用户表只有几万数据，所以使用的算法是：直接取出按连续签到次数排序好的用户ID数组，然后查找用户ID在其中的index，再加1即可。数据量不大的情况下，这么做还好。不过随着时间推移，现在有了40w的用户数据……

每天Mysql慢查询中，这块的sql都刷屏，所以决定对其进行优化。

未改动之前的代码如下：

{% highlight ruby %}
  ranks = Member.order('serial_checkins desc, id desc').pluck(:id)
  @rank = ranks.index(current_member.id) + 1
{% endhighlight %}

Mysql木有rank函数，而用变量的方式模拟rank函数需要嵌套子查询，效能会更烂。所以最先想到的是对ranks数组改用二分查找，于是不假思索的操刀就开干了。

{% highlight ruby %}
  ranks = Member.order('serial_checkins desc, id desc').pluck(:id)
  @rank = ranks.bsearch {|n| n >= current_member.id} + 1
{% endhighlight %}

跑了下测试，发现没通过……琢磨了一下，想到原因是：这个用户ID数组必然是乱序的，所以不能使用二分查找OTL

琢磨了几分钟，想到了一个曲线救国的方案：

{% highlight ruby %}
  # count所有连续签到次数高于当前用户的用户总数
  upper_count = Member.where('serial_checkins > ?', current_member.serial_checkins).count
  # 取出所有连续签到次数和当前用户一样的，升序排列后的用户ID组
  sblings = Member.where(serial_checkins: current_member.serial_checkins).order(id: :asc).pluck(:id)
  # 对这个ID组进行二分查找，找到当前用户的index值
  sblings_count = sblings.bsearch_index {|n| n >= current_member.id} + 1
  # 两个count相加……
  @rank = upper_count + sblings_count
{% endhighlight %}

因为使用这个功能频繁的人大多处于签到排行头部集团，所以这一算法取出的2项数据量都很小，速度会非常快。
如果用户处于非头部集团，那么同排位的人可能很多，用二分查找和原来比较也能大大提升效率。:w
