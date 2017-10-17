---
layout: post
title: "整合SendCloud和Postfix"
date: 2017-10-17 22:30:31
category: linux
---

安装postfix作为邮件发送服务。

{% highlight bash %}
apt-get install postfix
{% endhighlight %}

安装libsasl2验证模块，以让postfix支持SASL验证。

{% highlight bash %}
apt-get install libsasl2-modules
{% endhighlight %}

编辑postfix的配置文件（一般位于/etc/postfix/main.cf），加入以下内容：

{% highlight bash %}
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_sasl_tls_security_options = noanonymous
smtp_tls_security_level = encrypt
header_size_limit = 4096000
relayhost = [smtp587.sendcloud.net]:587
{% endhighlight %}

创建登录验证文件并编辑加入sendCloud相关登录信息。

{% highlight bash %}
touch /etc/postfix/sasl_passwd
vi /etc/postfix/sasl_passwd
#内容： [smtp587.sendcloud.net]:587 你的app_id:你的app_key
{% endhighlight %}

调整验证文件的权限，并用postmap校验&启用验证文件。

{% highlight bash %}
chmod 600 /etc/postfix/sasl_passwd
postmap /etc/postfix/sasl_passwd
{% endhighlight %}

重启postfix服务

{% highlight bash %}
systemctl restart postfix
{% endhighlight %}

至此，postfix用SendCloud发生邮件即配置成功。可以登录本地telnet，检查下发送邮件是否成功。

{% highlight bash %}
telnet localhost 25

ehlo yourdomain.com
(输入后会返回一大段250响应)
mail from: root@yourdomain.com
rcpt to: receiver@xxx.com
data
此处随意输入些邮件内容
. (结束正文内容)
{% endhighlight %}

之后会返回类似 "250 2.0.0 Ok: queued as 6E414C4643A" 的信息。

如果配置没问题的话，你的email的垃圾箱应该会收到一封信……
