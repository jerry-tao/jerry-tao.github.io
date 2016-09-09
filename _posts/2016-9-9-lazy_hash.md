---
layout: post
title: "使用延迟加载来优化redis请求"
date: 2016-9-8 12:47:06
categories: Redis
image: /assets/images/old-1130743_1280.jpg
---

## Background

我们系统允许客户自己对页面进行编辑，基于liquid实现的，提供了一大堆变量以供在页面上显示。

因为不确定客户会使用哪些变量，所以在渲染liquid之前就要把提供的对象都从redis里面提取出来准备好，但实际上也许只使用了其中一部分的一部分。而且由于每个页面都需要加载所有变量，对redis来说非常不友好（通信量过大）。

## Round 1

一开始的思路是查看liquid用了哪些变量，然后只提取使用的变量，但是这样有两个问题：

- 不得不提前解析一次liquid，性能影响很大。
- 除了liquid本身的语法，我们还自定义了一些其他的用法，还有页面之间的include，解析难度也比较大。

放弃。

## Round 2

仔细研究了一下liquid的render，实际上对变量的读取就是通过hash的key来实现的，并且通过测试修改hash.[]方法就可以做到自由控制变量的入口了。

所以改成了以lazy_hash的形式来读取redis，需要什么取什么。

```ruby
class LazyHash < Hash
  attr_accessor :default_key
  def initialize(key)
    default_key = key
  end

  def [](key)
    result = super(key)
    return result if result!=nil
    result = $redis.hget(default_key, key)
    self.[]=(key, result)
  end
end  
```

改动后可以达到需要key才去redis里面读，多次使用同一个key不需要每次都去读的效果。

## Round 3


但是在实际中发现liquid并没有渲染，binding进liquid发现liquid会使用key?对hash先判断一下key是否存在。于是继续修改如下

```ruby
# 追加方法
  def key?(key)
    self.[](key) != nil
  end
```

到此基本已经达到目的了，不过还有一个小问题。

## Round 4

那就是用户在编辑liquid的时候可能为了测试会直接输出对象，例如：

```
{ global_setting }

{ global_setting | json }
```

为了应对这种情况还是要提供一些load!方法的：


```ruby
  def load!
    merge!($redis.hgetall(default_key))
  end
  def inspect
    load!
    super
  end

  def to_s
    load!
    super
  end

  def to_json
    load!
    super
  end
```

至此基本优化完成，测试效果是redis的请求数量大概提升了一倍，但是总的带宽降低5倍以上。

这个其实没有通用性，只能说具体问题具体分析，只是作为一种优化思路提供。