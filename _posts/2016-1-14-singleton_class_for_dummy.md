---
layout: post
title: "Singleton Class简明教程"
date: 2016-1-14 14:47:06
categories: ruby
image: /assets/images/old-1130743_1280.jpg
---

也许你经常听说singleton class，但是并没有搞清楚它到底是什么，我希望能用尽量简单的语言说明白。

首先我们必须认识到两件事：

- 第一，我们常说的class和module，也是ojbect，例如`String.class #=> Class`。
- 第二，class是一种用来生成object的模板。

有了这两个基础知识，实际上问题已经简化为singleton class对于object来说是什么。

对于每一个object来说，会有一个为它提供实例方法的class，即:

```ruby
person = Person.new
# person里面的实例方法均来自Person
```

理解了object跟class的关系之后，singleton就可以用一句话来说明：**singleton class是只属于一个对象的class**。

所以我们可以打开这个class在里面添加实例方法，即可被这个object使用。跟打开这个object的class添加实例方法的区别在于：在singleton class里面添加的实例方法仅可被该object使用，而再次打开这个class添加的实例方法则可以被所有这个class的object使用。

```ruby
person = Person.new
person2 = Person.new

# 打开Person添加say方法
class Person
  def say
    puts 'Say my name!'
  end
end

person.say # => 'Say my name!'
person2.say # => 'Say my name!'

# 打开person的singleton class为person添加say_hi方法
class << person
  def say_hi
    puts 'Say hi!'
  end
end
person.say_hi # => Say hi!
person2.say_hi # Error
```

如果上面的可以理解，我们再来看下面的：

```ruby
# 打开Person的singleton class为Person添加实例方法
class << Person
  def say_word
    puts "say_world"
end

# 在Person内打开Person的singleton class
class Person
  class << self # 此时self就是Person 跟上面打开person的singleton class并无区别
    def say_hello
      puts 'Say hello!'
    end
  end
end
```

如果以上的你都可以理解，那恭喜你已经掌握了一半关于singleton class的知识了，然后让我们来看一些不一样的东西：**实际上，我们所说的类方法就是定义在singleton class里面的**。

```ruby
# 定义一个类方法
class Person
  def self.say_goodbye
     puts 'Say goodbye!'
  end
end
Person.singleton_methods                                         
# => [:say_goodbye] 如果还不信可以直接看singleton的instance_method
Person.singleton_class.instance_methods                          
# => [:say_goodbye,...]

# 另一种方式
def Person.say_something
  puts 'Say something'
end
Person.singleton_methods  
# => [:say_goodbye, :say_something]

# 定义在object上，实际上跟上面并没有区别
def person.say_anything
  puts 'Say anything!'
end
```


除此之外，我们还可以继续在singleton class上继续定义singleton method, 因为singleton class也只是class的一个实例，所以跟你在其他object上定义singleton class并无任何区别。

Singleton class的存在提供了很多灵活性，例如我们常见的类方法或者模块方法，都是通过这种机制直接实现的，或者是在rails中的class_attribute，也是通过不断的修改singleton method来实现的。