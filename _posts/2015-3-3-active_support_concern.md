---
layout: post
title: "ActiveSupport Concern"
date: 2015-3-3 11:47:06
categories: sources
image: /assets/images/glasses.jpg

---

其实很早之前就写过concern的blog，但是貌似没有保存，为了保持active_support完整性，再整理一次。

其实concern就是简化了mixin的写法，让mixin写起来更舒服些，看下面的demo。

```ruby
#without concern
module A
  def self.included(base)
    #Call class method here.
    # => base.has_many :user # Just for example
    #extend class methods here.
    base.extend ClassMethods
  end

  module ClassMethods
    def hello
      p 'hello'
    end
  end
end

class Team
  include A
end

Team.hello

#with concern
module A
  extend ActiveSupport::Concern
  included do |base|
    #Call class method here.
    # has_many :user # Just for example
    #extend class methods here.
  end

  class_methods do # Or use module ClassMethods
    def hello
      p 'hello'
    end
  end
end

class Team
  include A
end

Team.hello
```

关于这个的使用说明和原理介绍实在是太多了，使用extended和append_features两个hook来实现扩展。

整理一下ruby里常见的hook吧。

### Method-related hooks

```ruby
method_missing
method_added
singleton_method_added
method_removed
singleton_method_removed
method_undefined
singleton_method_undefined
```

### Class & Module Hooks

```ruby
inherited
append_features
included
extend_object
extended
initialize_copy
const_missing
```

### Marshalling Hooks

```ruby
marshal_dump
marshal_load
```

### Coercion Hooks

```
coerce
induced_from
to_xxx
```
