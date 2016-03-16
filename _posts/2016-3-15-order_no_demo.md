---
layout: post
title: "高并发订单号池"
date: 2016-3-15 20:47:06
categories: 脑洞
image: /assets/images/old-1130743_1280.jpg

---


最近回答了一个关于订单号生成的问题，然后脑洞大开决定用ruby实现一个支持高并发的订单号池。

订单号池不算很复杂，就是个简单的生产者消费者模型，看图说话。

![pc](/assets/images/pc.png)

worker里面一共有两个：order\_check和order\_no。

其中order\_check设为周期任务，时间随意了，反正就是定期检查一下redis里的order\_no还剩多少，不够了就触发一下order\_no\_worker。

```ruby
# order_check
def perform
  count = $redis.scard('order_no')
  if count < 1000
    OrderNoWorker.perform_async
  end
end
```

order\_no用来生成order\_no并扔进redis，生成order\_no我写的比较简单：

```ruby
# order_no
def order_no
  now = Time.now
  Time.now.strftime('%Y%m%d') + now.usec.to_s + ("%05d" % rand(9999))
end
```

redis的存储用的是set，保证唯一即可。

重点还是消费啊，一开始写了个sinatra，

```ruby
require 'sinatra'
require 'json'

get '/' do
  order_no = $redis.spop('order_no')
  order_no.nil? ? {error:'There is no order_no current.'}.to_json : {order_no: order_no}.to_json
end
```

ab测试

```
ab -c100 -n15000 http://localhost:9292/
# HTML transferred:       508434 bytes
# Requests per second:    1020.89 [#/sec] (mean)
# Time per request:       97.953 [ms] (mean)
```


测试效果呵呵呵，1000req/s，虽然只是用rack启动下也真的是接受不能。

直接换成rack 来启动个object。

```
require 'json'
class OrderNoService
  def order_no
    order_no = $redis.spop('order_no')
    order_no.nil? ? {error:'There is no order_no current.'}.to_json : {order_no: order_no}.to_json
  end
  def call(env)
    ['200', {'Content-Type' => 'application/json'}, [order_no]]
  end
end
```

ab测试

```
ab -c100 -n15000 http://localhost:9292/
# HTML transferred:       508423 bytes
# Requests per second:    1648.45 [#/sec] (mean)
# Time per request:       60.663 [ms] (mean)
```

比上面好点，但是跟预期差了一个数量级啊。

恰好还记得这篇文章[25000 req/s with mruby](http://lucaguidi.com/2015/12/09/25000-requests-per-second-for-rack-json-api-with-mruby.html)，然后发现mruby已经有[ngx_mruby](https://github.com/matsumoto-r/ngx_mruby)了，拿来试下。

崩溃的发现，mruby-redis还没有支持spop，自己加上之后再次测试：

```
ab -c100 -n10000 http://127.0.0.1:8080/mruby
# HTML transferred:       340000 bytes
# Requests per second:    14339.08 [#/sec] (mean)
# Time per request:       6.974 [ms] (mean)
```

终于达到了预期的数量级，测试机只是i5的配置，如果换成服务器，再好好配置一下nginx，性能应该好很多。完整代码在[order\_no\_demo](https://github.com/jerry-tao/order_no_demo)

只是个不完整的demo，真实世界要比这复杂的多，下面加些FAQ。

## FAQ：


**这个生成订单号的规则没问题么？**

有问题，如果当日真的check到不足二次生成的时候是可能会重复的，因为期间set已经被pop出去并且纳秒是很容易重复的，没有使用完整的时间戳是因为太长了，而且仅仅是为了写个demo。

解决方案包括如下几个：

- 加计数器控制不同批次。
- 把数据结构换成hash不pop修改状态。
- 换一种生成规则。
- 集群的时候记得加机器码来确保唯一。

**这东西有毛线的实际意义？**

没有，真的没有。

**为什么sinatra和rack的测试都是ab -c100 -n15000，mruby的时-c100 -n10000?**

因为15000在我机器上block了，不确认是我机器原因还是mruby-redis的原因，已提issue，目前不建议在生产环境中使用ngx_mruby。

**ab测试准么?**

当然不准啊，不过一整个数量级的差距还是很明显的。
