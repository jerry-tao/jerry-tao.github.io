---
layout: post
title: "Ruby Hook Methods"
date: 2016-1-13 14:47:06
categories: ruby
image: /assets/images/old-1130743_1280.jpg
---

Hook Methods一般译为钩子方法，可以理解为当触发了某个条件执行的方法。例如我们常见的method\_missing和上一章里介绍的const\_missing。

**Method相关**

- respond\_to\_missing?: 当尝试查看一个missing的方法时执行。
- method_missing: 当调用一个不存在的方法时执行。
- method_added: 当定义一个方法时执行
- method_removed: 当一个方法被移除时执行
- singleton\_method\_added: 当添加一个singleton方法时执行。
- singleton_method_removed: 当一个singleton方法被移除时执行。
- method\_undefined: 当一个方法被undefined的时候执行，undef和remove的区别在于当你在子类中你可以undef掉从父类继承而来的方法，而remove则不可以删除定义在父类中的方法。在父类中无论是使用undef和remove子类都无法继续使用这些方法。
- singleton\_method\_undefined: 当一个singleton方法被undefined时刻执行。

示例代码：

```ruby
# respond_to_missing?
class Base
  def respond_to_missing?(sym, *args)
    true
  end
end
Base.new.respond_to?(:missing_method)
# 这个跟重写respond_to?的区别主要在于使用下面这个方法
Base.new.method(:missing_method)
# => #<Method: Base#missing_method>

# method_missing
class Base
  def method_missing(method_name)
    puts method_name
  end
end
Base.new.missing_method
# => missing_method

# method_added

class Base
  def self.method_added(method_name)
    puts method_name
  end

  def missing_method
  end
end
# => missing_method
Base.class_eval { def missing_method2;end }
# => missing_method2

# method_removed
class Base
  def self.method_removed(method_name)
    puts method_name
  end
end    
Base.class_eval { remove_method :missing_method }
# => missing_method

# method_undefined
class Base
  def self.method_undefined(method_name)
    puts method_name
  end
end    
Base.class_eval { undef_method :missing_method }
# => missing_method

# singleton_method_added/removed/undefined
class Base
  def self.singleton_method_added(method_name)
    puts method_name
  end

  def self.missing_method
  end

  def Base.missing_method1
  end
end
def Base.missing_method2
end

# => singleton_method_added
# => missing_method
# => missing_method1
# => missing_method2
```
**Class/Object相关**

- inherited: 当被继承时执行。
- initialize_copy: 当调用initialize\_clone和initialize\_dup时执行。
- initialize_dup: 当调用dup，返回dup结果前时执行。
- initialize_clone: 当调用clone，进行frozen判断和返回clone结果之前执行。

介绍这几个方法之前我们先要了解一下dup和clone的区别，主要有以下两点：

- dup不会复制frozen状态，clone会。
- dup不会复制singleton class，clone会。

```ruby
# inherited
class Base
  def self.inherited(sub)
    puts sub.name
  end
end
class Sub < Base
end

# => Sub

# initialize_*
class Base
  def initialize_copy(other)
    puts 'initialize_copy'
    super
  end
  def initialize_dup(other)
    puts 'initialize_dup'
    super
  end
  def initialize_clone(other)
    puts 'initialize_clone'
    super
  end
end
Base.new.dup    
# => initialize_dup                  
# ＝> initialize_copy
Base.new.clone                      
# => initialize_copy
# => initialize_clone
```

**Modules相关**

- included: 当include时执行。
- append_features: 当include时执行。
- prepended: 2.0新增prepend的时候执行。
- prepend\_features：跟上面的append\_feature可以一同理解。
- extend_object: 当被extend时执行。
- extended: 当被extend时执行。
- const_missing: 当有常量丢失时执行。

首先来说，对于include，prepend这两者和extend的关系很容易区分。include和prepend的区别在于对祖先链的修改方式，进一步影响方法的查找流程。

参考下面的例子：

```ruby
module M
  def hello
    puts 'Hello from M'
    super
  end
end

module N
  def hello
    puts 'Hello from N'
  end
end

class C
  include N
  prepend M
  def hello
    puts 'hello from C'
    super
  end
end
C.new.hello
#=> Hello from M
#=> hello from C
#=> Hello from N
```

其中included和append\_features是一组，prepended和prepend\_features是一组，extended和extend\_object是一组。

以included和append\_features来说，append_features这个方法是真正把module里的代码混入到include此module的地方，而included仅仅是在完成混入之后执行的一个回调，下面是一个伪示例:

```ruby
module M
  def self.append_features(mod)
    mod.class_eval {def new_method;puts 'say hi!';end}
  end

  def self.included(mod)
    puts 'included'
    mod.new.new_method
  end
end
```

对于prepend和extend两组来说也是如此。

**Marshal相关**

marshal\_dump: 当Marshal.dump(@obj)时执行。
marshal\_load: 当Marshal.load(@obj)时执行。

```ruby
class C
  attr_accessor :arrs, :d_arrs
  def marshal_dump
    Marshal.dump(arrs)
  end
  
  def marshal_load(data)
    self.d_attrs = Marshal.load(data)
  end
end

k = C.new
k.arrs = [1,2,3,4,5]
k2 = Marshal.load(Marshal.dump(k))
k2.d_arrs 
# => [1,2,3,4,5]
k2.arrs 
# => nil
```

**Coercion相关**

- coerce: 当使用一个二元操作符的时候会在第二个参数上触发，当我们重载运算符的时候很有用。
- to\_*系列方法: 这些方法虽然看起来不像，但其实很多时候都会被触发：

```ruby
# coerce
class C
  def coerce(n)
    puts n
    [5,n]
  end
  def to_s
    "CCCC"
  end
  def to_int
    3
  end
  def to_proc
    Proc.new {|i| puts i}
  end
end
3*C.new
#=> 3
#=> 15
"#{C.new}" #=> CCCC 会触发to_s
[1,2,3,4,5][C.new]                                               
# => 4 触发to_int
[1,2,3,4,5].each(&C.new) # to_proc
# => 1
# => 2
# => 3
# => 4
# => 5
```