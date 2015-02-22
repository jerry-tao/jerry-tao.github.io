---
layout: post
title: "ActiveSupport Aliasing"
date: 2015-2-22 11:47:06
categories: sources
image: /assets/images/glasses.jpg

---

Aliasing(`active_support/lib/active_support/core_ext/module/aliasing.rb`)是`active_support`中对module的一个扩展，这里面就定义里两个方法, `alias_method_chain`和`alias_attribute`。


## alias\_method_chain

这个方法有点类似轻量级的AOP, 其实这个功能ruby本身就可以通过alias和alias_method做到，这个扩展只不过让这个功能用起来更简单一些。

先看一下效果吧

```ruby
require 'active_support'
require 'active_support/core_ext'
class Demo
  def hello
    p 'hello!'
  end

  def hello_with_name
    p 'Before say hello!'
    name = 'Alice.'
    p 'Alice'
    hello_without_name
    p 'After say hello!'
  end

  alias_method_chain :hello, :name
end

Demo.new.hello
#"Before say hello!"
#"Alice"
#"hello!"
#"After say hello!"
```

通过上面的例子可以看出，通过使用alias_method_chain, 你可以很轻松的在某个方法执行前后加入你想自定义执行的内容。

使用的方式应该也可以看出来，你需要定义一个`原方法名_with_用于alias_method_chain的名字`的方法，然后在这个方法内部可以通过`原方法名_without_用于alias_method_chain的名字`来调用原来的方法。

比如我最近使用的paranoid这个gem，我想给每个model都加上`act_as_paranoid`来把每个model都变成软删除的效果。

并不能直接打开ActiveRecord::Base去monkey patch，因为这个会得不到table_name, 所以：

```ruby
#config/initializers/paranoid.rb
ActiveRecord::Base.module_eval do
  class << self
    def inherited_with_paranoid(subclass)
      skip_models = %w(schema_migrations versions activities)

      inherited_without_paranoid(subclass)

      table_name = subclass.table_name

      if !skip_models.include?(table_name) && table_name.present?
        subclass.send(:acts_as_paranoid)
      end
    end

    alias_method_chain :inherited, :paranoid
  end
end
```

你可以打开这个源码直接看看这个方法，核心就是alias_method的使用。

## alias_attribute

这个方法相对而言就简单的多了。只是简单的定义三个新的attr, attr=, attr?方法。

```ruby
class Post < ActiveRecord::Base
  #假设有个name属性
  alias_attribute :title, :name
end
post = Post.find(1)
expect(post.title).to eq(post.name)
post.title = 'New Title'
expect(post.title).to eq(post.name)
expect(post.title?).to eq(post.name?)
# 就是说你现在可以使用post.title, post.title=, post.title? 来代替原来的post.name, post.name=, post.name?
```
