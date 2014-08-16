---
layout: post
title: "解决因为rspec升级而引起spork报告uninitialized constant RSpec::Core::CommandLine错误"
date: 2014-08-16 13:35:31
category: ruby
---

前几天同事更新了一下项目的Gemfile，从Git拉回来之后随手bundle了一下……
结果今日要写新东西，启动spork后开始跑测试，结果得到下面的错误：

{% highlight bash %}
Exception encountered: #<NameError: uninitialized constant RSpec::Core::CommandLine>
{% endhighlight %}

开始以为是spec_helper里少了包含文件，检查了下发现没问题。一头雾水的去google了一下，发现原来是rspec升级导致的：rspec3里将类RSpec::Core::CommandLine合并进了RSpec::Core::Runner里。但是疏于维护的spork并没有跟进更新，所以导致spork启动时无法加载该类。

github上有人给出了[解决方案](https://github.com/manafire/spork/commit/38c79dcedb246daacbadb9f18d09f50cc837de51#diff-937afaa19ccfee172d722a05112a7c6fL6)

修改spork的源码，找到Gem所在路径下的 /lib/spork/test_framework/rspec.rb 文件，将其替换为如下内容即可。

```ruby
# 增加require该文件
require 'rspec/core/version'
class Spork::TestFramework::RSpec < Spork::TestFramework
  DEFAULT_PORT = 8989
  HELPER_FILE = File.join(Dir.pwd, "spec/spec_helper.rb")

  def run_tests(argv, stderr, stdout)
    if rspec1?
      ::Spec::Runner::CommandLine.run(
        ::Spec::Runner::OptionParser.parse(argv, stderr, stdout)
      )
    # 增加判断是否repec3的分支
    elsif rspec3?
      options = ::RSpec::Core::ConfigurationOptions.new(argv)
      ::RSpec::Core::Runner.new(options).run(stderr, stdout)
    else
      ::RSpec::Core::CommandLine.new(argv).run(stderr, stdout)
    end
  end

  # 增加rspec3的判断方法
  def rspec3?
    return false if !defined?(::RSpec::Core::Version::STRING)
    ::RSpec::Core::Version::STRING =~ /^3\./
  end

  def rspec1?
    defined?(Spec) && !defined?(RSpec)
  end
end

```
