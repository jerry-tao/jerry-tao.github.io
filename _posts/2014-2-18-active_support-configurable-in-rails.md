---
layout: post
title: "ActiveSupport::Configurable"
date: 2014-02-18 11:47:06
categories: Rails Sources
image: /assets/images/post.jpg
---

Configurable是active_support里面让一个类可以实现configurable的扩展，先看一个demo。

{% highlight ruby %}
class Setting
  include ActiveSupport::Configurable
end

Setting.config.send_mail=true
Setting.config
#{:send_mail=>true}

admin.setting = Setting.new
admin.setting.config
#{}
admin.setting.config.send_mail
#true

admin.setting.config.send_mail=false
admin.setting.config
#{:send_mail=>false}
admin.setting.config.send_mail
#false
{% endhighlight %}

从上面的例子可以看出，config既可以作为一个类方法使用也可以为实例添加，实例默认是跟class一样的config，不过也可以自定义并且不影响class的config。

实现这部分的源码：
{% highlight ruby %}
  module Configurable
    extend ActiveSupport::Concern

    class Configuration < ActiveSupport::InheritableOptions
     #...
    end

    module ClassMethods
      def config
        @_config ||= if respond_to?(:superclass) && superclass.respond_to?(:config)
          superclass.config.inheritable_copy
        else
          # create a new "anonymous" class that will host the compiled reader methods
          Class.new(Configuration).new
        end
      end
    end


    def config
      @_config ||= self.class.config.inheritable_copy
    end
  end
{% endhighlight %}
Configuration是继承自InheritableOptions的，也稍微看一眼这个类
{% highlight ruby %}
  class InheritableOptions < OrderedOptions
    def initialize(parent = nil)
      if parent.kind_of?(OrderedOptions)
        # use the faster _get when dealing with OrderedOptions
        super() { |h,k| parent._get(k) }
      elsif parent
        super() { |h,k| parent[k] }
      else
        super()
      end
    end

    def inheritable_copy
      self.class.new(self)
    end
  end
{% endhighlight %}
在这里可以看到，在configurable里面有两个叫做config的方法，一个是class method 一个是instance method(这不是废话么)。

当调用instance的时候，会先检测@_config变量，如果没有则copy一份class的@_config。

这部分copy的代码也很精巧，这个configuration一直继承自hash，当copy的时候并没有直接复制，而是传给了一个block给hash。

一段简单的带scope的hash的实例：
{% highlight ruby %}
a = Hash.new() {|h,k| h[k]="key:#{k}"}
a[:a]
#=> key:a
{% endhighlight %}

再来梳理一遍这个流程。

当调用class的config的时候，会先生成一个configuration实例，一直继承自OrderedOptions，当调用instance的时候，会通过InheritableOptions来调用class里面的configuration。
