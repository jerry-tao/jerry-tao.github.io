---
layout: post
title: "ActiveSupport Notifications"
date: 2015-2-23 11:47:06
categories: sources
image: /assets/images/glasses.jpg

---

ActiveSupport::Notifications实现了一个轻量级的发布订阅系统，可以用来解耦我们的业务逻辑，主要涉及的文件是下面两个：

- lib/active_support/notifications.rb
- lib/active_support/subscriber.rb
- lib/active_support/notifications/fanout.rb
- lib/active_support/notifications/instrumenter.rb

先来看一个简单的使用例子吧：

```ruby
require 'active_support/notifications'
# 注册一个订阅者
ActiveSupport::Notifications.subscribe('notifications.hello') do |name, start, finish, id, args|
    p "#{args[:name]} said Hello by calling #{name}"
end
# 发布一条消息
ActiveSupport::Notifications.instrument('notifications.hello',  name:'Jerry' )
```

然后通过使用subscriber我们可以为我们的订阅者实现一个namespace：

```ruby
require 'active_support/notifications'
require 'active_support/subscriber'
class EventSubscriber < ActiveSupport::Subscriber
  def say_hello(event)
    p event
  end
end
EventSubscriber.attach_to :namespace_demo # 把我们的subscriber注册到namespace

ActiveSupport::Notifications.instrument('say_hello.namespace_demo',  options:'Hello' )
```

在这个例子中，namespace_demo就是我们event的namespace，方法名say_hello就是事件的名称。

考虑更加真实一点的应用场景，就是当我们使用engine来解耦的时候，在engine和application直接进行通信。可以看一下这篇文章：[Using events to decouple Rails applications](https://redbooth.com/engineering/patterns/using-events-decouple-rails-applications)
