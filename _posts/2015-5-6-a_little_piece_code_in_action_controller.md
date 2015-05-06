---
layout: post
title: "A Little Piece Code in ActionController"
date: 2015-5-6 11:47:06
categories: sources
image: /assets/images/chemistry-740453_1280.jpg

---

我们都知道当使用rails render的时候可以render各种格式的响应，例如下面响应json，xml。

```ruby
render json: @obj
render xml: @obj
```

并且，我们是可以自己定义自己的render格式的，比如 Crafting Rails 4 Applications 里介绍的`render pdf: @content`。

下面这个代码简单重现了这个过程是怎么实现的。 比较好玩的地方就是关于'如何来保证扩展性'这一点的实现方式。

```ruby
renders = Set.new

# 使用这个方法可以添加一种新的格式 ActionController::Renderers.add
def add(key, &block)
  define_method("_render_option_#{key}", &block)
  renders <<  key.to_sym
end

# 如何使用add方法
add :json do |json,options|
  # 真正render json的代码。
end

# render时的处理
def render(options)
  renders.each do |name|
    if options.key? name
      send("_render_option_#{name}", options.delete(name), options)
    end
  end
end
```
在这里还发现了另一段非常好玩的代码：

```ruby
module Renderers
  module All
    include Renderers
  end
end
# 简化版
module A
  module B
    include A
  end
end
```
仔细想下这个逻辑还算蛮好玩的。

这里的代码可以在这里看到 https://www.omniref.com/ruby/gems/actionpack/4.2.1/symbols/ActionController::Renderers?d=549715196&n=3
