---
layout: post
title: "ActiveSupport PerThreadRegistry：Get Controller Everywhere"
date: 2015-2-27 11:47:06
categories: sources
image: /assets/images/glasses.jpg

---

经常看到这样的问题 'How can i get current_user/controller/action/request in models?'。并且这种问题的解决方案一般来说都很一致：存进Thread.current[]里。

(虽然我个人并不喜欢这种做法，不过很多时候这样做确实很简单，并且说实话我并没有找到更好的办法，what happend in controller, should stay in controller.)

然后我们看一下使用PerThreadRegistry来存储的方法。本质上来说，存储的地方仍然是Thread.current[], 只是封装了一下然后加了一个namespace。

## 使用

先看一个简单的用法：

```ruby
class DemoRegistry
  extend ActiveSupport::PerThreadRegistry

  attr_accessor :user
end

DemoRegistry.user = 'Jerry'
DemoRegistry.user
#=>'Jerry'
Thread.current['DemoRegistry'].user == DemoRegistry.user
#=> true
```

到这里其实很明显如何把current_name之类的变量存储在thread里了。

看一下存储current_user的例子：

```ruby
class PostsController < ApplicationController
  before_action :set_current_user

  private

  def set_current_user
    UserRegistry.user = current_user
  end
end

class Post<ActiveRecord::Base
  after_save :set_user
  def set_user
    UserRegistry.user #这里就能取出来了
  end
end
```

## 源码

```ruby
module PerThreadRegistry
  def self.extended(object)
    object.instance_variable_set '@per_thread_registry_key', object.name.freeze # 用object的名字做namespace
  end

  def instance
    Thread.current[@per_thread_registry_key] ||= new # 在一个线程内单例
  end

  protected
  def method_missing(name, *args, &block) # :nodoc:
    # Caches the method definition as a singleton method of the receiver.
    define_singleton_method(name) do |*a, &b|
      instance.public_send(name, *a, &b) # 调用instance的mehtod
    end

    send(name, *args, &block)
  end
end
```

## 补充

个人并不喜欢通过这种做法来存储对应的内容，不过貌似如果想在model里得到这些内容也只有传参和存储在Thread两种方法。所以如果想做到what happend in controller, should stay in controller. 就不得不把部分逻辑挪到controller里。

不过有一点还是OK的，实际上当你存储controller在Thread里，你只存储了一个引用，并没有复制一个controller，所在资源上应该还可以接受。

<!--关于使用完成之后，是否需要'清空' 这个Namespace，这个需要看两点，如果是线程池的话，那么你可能需有手动清空一下，否则如果没有新建线程那这个变量会存留知道下次使用，如果不是线程池（貌似rails没有使用线程池），那么不需要手动清空。-->
