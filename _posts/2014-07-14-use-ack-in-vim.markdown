---
layout: post
title: "Vim中使用Ack替代grep进行搜索"
date: 2014-07-14 11:40:41
category: vim
---

用grep在Vim下搜索太蛋疼了，于是换用了大家喜闻乐见的Ack。由于用的是Debian，所以安装过程和红帽系的略有不同，记录如下：

因为Debian下有另外一个包也叫ack，所以ack改名为了ack-grep。直接无脑aptitude安装。

{% highlight bash %}
aptitude install ack-grep
{% endhighlight %}

安装完了直接

{% highlight bash %}
Bundle 'mileszs/ack.vim'
{% endhighlight %}

完ack.vim后，发现在vim里打Ack命令搜索会出错，提示："Unknown option: s"。先google了一下，没结果，于是去Github看了下，发现原来需要用2.0以上版本的Ack……

把apt装的1.9的卸载后，去官网拉了一个2.12的：

{% highlight bash %}
curl http://beyondgrep.com/ack-2.12-single-file > /usr/bin/ack && chmod 0755 !#:3
{% endhighlight %}

再进Vim，终于能用了。

[Ack官方文档](http://beyondgrep.com/documentation/)

Vim中按下:后，输入"Ack [参数] 需要匹配的字串 [路径]"后，回车，即可看到搜索结果。

随手写的几个简单的示例：

查找当前文件夹下ruby类型文件中含有类似"products[此处是key值]"字段的文件

{% highlight bash %}
Ack --type=ruby 'products\[[a-z:]+\]' ./
{% endhighlight %}

（结果太多时，可以加入--break参数在文件间加入间隔）

因为ack使用Perl的正则，所以当你搜索字串中含有正则的匹配符时，例如下面这样：

{% highlight bash %}
Ack 'products[]'
{% endhighlight %}

你将会收到一个错误："Invalid regex 'products[]':"

此时可以使用-Q参数来忽略正则元字符：

{% highlight bash %}
Ack -Q 'products[]'
{% endhighlight %}
