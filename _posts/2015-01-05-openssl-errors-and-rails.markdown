---
layout: post
title: "解决rails中遇到的OpenSSL::SSL::SSLError"
date: 2015-01-05 10:30:15
category: rails
---

# 背景概述：

之前生产环境的服务器一直会不定期（几个月）发生负载过高的问题一直没能解决，该问题在2014的最后一天又不期而至OTL 于是Boss决议将生产环境移至另一台备用服务器。

操刀完成后，悲剧发生：微信登录无法使用了。查看了下错误日志，发现一个没见过的神奇错误刷屏……

{% highlight bash %}
  openssl::ssl::sslerror: ssl_connect returned=1 errno=0 state=sslv3 read server certificate b: certificate verify failed
{% endhighlight %}

# 解决过程：

google一顿，找到[一篇博文](http://railsapps.github.io/openssl-certificate-verify-failed.html)，其中解释错误原因是因为服务器上的SSL证书过期。不过按照其给出的方法操作后，问题依旧……

{% highlight bash %}
$ rvm -v
# rvm 1.19.1 (stable)
$ rvm osx-ssl-certs status all
# Certificates for...
$ rvm osx-ssl-certs update all
# Updating certificates...
{% endhighlight %}

比对了一下新/旧生产环境的openssl版本，发现不一致。

{% highlight bash %}
openssl version
{% endhighlight %}

由于琢磨着是不是ruby扩展里的ssl版本和ruby版本不兼容导致的，就重新安装了一下ruby和openssl扩展，结果问题神奇的解决了。

{% highlight bash %}
rvm pkg install openssl
rvm reinstall 2.0.0
{% endhighlight %}
