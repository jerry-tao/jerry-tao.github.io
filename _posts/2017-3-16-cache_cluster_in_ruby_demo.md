---
layout: post
title: "杂七杂八的缓存"
date: 2017-3-16 20:47:06
categories: Fun
image: /assets/images/stopwatch-2061845_1280.jpg
---

## 1. 缓存

缓存已经是常见的技术了，甚至可以说是必备的技术，缓存基本有两种主要的使用场景：

- 作为数据库的快速读缓冲

把常用的数据存储在缓存里用来加快响应速度，一般来说这种数据只是对RDBMS做一个镜像，当缓存中没有的时候我们还可以去db里获取原始内容。

- 作为传递数据的中介/存储一些可容忍丢失的数据

比如我们常用的sidekiq，就是用redis来存储任务的相关信息的。还有经常用来存储session，这种数据在一定程度上是允许不那么精确的。

注：缓存是一种unreliable的存储，这是在使用缓存的任何阶段都要明确的，以sidekiq为例，你不应该简单的数据添加进redis就不管了，应该确保redis给你明确返回了"OK"，不过即使redis返回了OK，如果在sidekiq读取之前崩溃了，相关数据并没有持久化依旧会导致任务的丢失，所以在作为传递数据使用缓存时需要确保后续的检查以及足够的retry机制。

## 2. 指标

我们往往会从下面三方面来评价一个缓存的使用情况：

- 命中率
- 一致性
- idletime

命中率是很重要的一点，例如我们有100个请求，其中只有20个命中了缓存，而其他的全部从数据库读取，这样我们的命中率就只有20%，不过让命中率达到100%是没有必要的，根据8：2原则，基本上80%左右就可以接受了。在redis的info命令keyspace\_hits和keyspace\_misses分别记录了访问命中和misses的数量。

一致性往往是被忽视的一点，很多人的习惯是先更新数据库后更新缓存，这种情况很容易导致缓存的数据已经过期了。

idletime是指我们把这个数据放入缓存中有多久没有使用过，在redis里可以通过OBJECT idletime key来检查。定期的清理一些idle key可以减少对硬件资源的占用。

注：redis的默认的达到最大使用内存是报错，可以通过设置来根据LRU消除还没有到期的key，这种配置也适用于减少idle key。

注：redis的info命令是并不是持久化的，也就是说记录的数据都是从启动至今为止的，如果你刚刚重启过redis，是无法得到info相关的信息的。

## 3. 集群基础

注：redis的多db理论上是用来解决key冲突的，特别是当你有不同的应用share同一个redis的时候可以更方便的对各自的信息进行管理，不过也要认识到，多个db并不会对读写性能有任何提升，如果在硬件资源允许的情况下，多个进程的解决方案要优于多db，同时很重要的事情是，有些redis集群方案是不支持多db的。

缓存的集群相对app来说稍微复杂了一点，不过其实核心问题主要是解决把哪个key放到哪个节点上，以及在节点有增删时对命中率和负载是否均衡的影响。

常见的就是通过key的hash值来取模，用来判断这个key应该放在哪个节点之上。例如我们现在有4个node：[c1,c2,c3,c4]，

```
"hello".hash%4
=> 3
```

这样就可以把hello放在c3上，不过这种简单取模的缺点也是很明显的，伸缩性非常不好。

假设我们现在有1w个key平均分散在4个节点之上，假设hash是从0~9999，当我们追加一个节点变成[c1,c2,c3,c4,c5]的时候，基本上只剩下20%的数据可以命中，而其他的80%都需要重建了。

所以更常见的做法是先虚拟出一定数量的节点，例如虚拟100个节点，然后对100取模，把10000个key依旧平均放在4个真实节点上。

这样当我们追加一个节点的时候，依旧能保持50%的命中率。

再进一步，如果我们把追加的节点放到中间，变成[c1,c2,c5,c3,c4]，这种情况下可以达到70%的命中率。所以我们可以把所有节点模拟成一个环（一致性hash算法），这样动态伸缩的时候基本都能维持70%的命中率。

环只是一个模拟的概念，可以进一步改进为使用虚拟节点+哈希表的实现，再增删节点的时候把虚拟节点平均分散到其他的节点之上。这样命中率可以保证在原节点数/新节点数。

一致性hash的缺点主要体现在复杂度上，需要O(N)的复杂度来查找节点。

解决了key的问题之后我们可以有两种方式来实现这个效果，一种是作为lib提供到各个使用redis的项目内，一种是在中间开发一个代理服务。

这两种方法各有各的优点，第一种的好处是你不需要一个额外的服务作为数据的中转，缺点是当面对动态的增删你可能需要做更多额外的工作，第二种的好处是所有的数据都经过一个中转便于统计和统一管理配置，缺点跟其他的代理服务器一样，代理服务器的性能可能成为整体的短板。

注：一致性Hash是比较常用的分布式策略之一，常使用的场景就是缓存集群。

## 4. 简单实现

下面是实现一个简单的缓存集群的两个要点

- 实现redis接口层
- 实现节点查找层

关于redis的接口层可以参考 《Redis的设计与实现》，下面是用ruby实现的一个模拟缓存集群的工作方式：

```
# 模拟一个Redis服务
class Redis
  attr_accessor :values
  def initialize(ip)
    @values = {}
  end
  def set(key, value)
    @values[key] = value
  end
  def get(key, value)
    @values[key]
  end
end

# 虚拟节点
class VNode
  attr_accessor :node
  def execute(*params)
    node.execute(*params)
  end
end

# 真实节点
class Node
  attr_accessor :vnodes,:ip, :redis
  
  def initialize(ip)
    @ip = ip
    @redis = Redis.new ip
    @vnodes = []
  end

  def execute(command, key, value)
    @redis.send(command,key,value)
  end
end

# 服务接口
class Server
  attr_accessor :nodes, :vnodes

  def initialize(ips)
    # 初始化2的16次方个虚拟节点
    @vnodes = (2**16).times.map { VNode.new }
    @nodes = ips.map {|ip| Node.new(ip)}
    build_vnodes(:initialize)
  end

  def add_node(ip)
    @new_node = Node.new(ip)
    build_vnodes(:add)
  end

  def remove_node(ip)
    @removed_node = @nodes.delete(@nodes.select {|item| item.ip == ip}.shift)
    build_vnodes(:remove)
  end

  def execute(command,key,value = nil)
    @vnodes[key.hash%@vnodes.size].execute(command,key, value)
  end

  def build_vnodes(key)
    case key
    when :initialize
      @vnodes.each_with_index do |vnode,index|
        @nodes[index % @nodes.size].vnodes << vnode
        vnode.node = @nodes[index % @nodes.size]
      end
    when :add
      move_size = @vnodes.size / @nodes.size - @vnodes.size / (@nodes.size + 1)
      @nodes.each do |node|
        @new_node.vnodes << node.vnodes.shift(move_size)
      end
      @new_node.vnodes.flatten!
      @new_node.vnodes.each {|vnode| vnode.node = @new_node}
      @nodes << @new_node 
      @new_node = nil
    when :remove
      @remove_node.vnodes.each_with_index do |vnode,index|
        @nodes[index % @nodes.size].vnodes << vnode
        vnode.node = @nodes[index % @nodes.size]
      end
      @remove_node = nil
    end
  end
end
```

然后来测试下10000个key的效果

```
require 'securerandom'
keys = 10000.times.map { SecureRandom.uuid }
ips = ['127.0.0.1','127.0.0.2', '127.0.0.3' , '127.0.0.4']
@server = Server.new ips
keys.each {|key| @server.execute 'set',key, true }
result = @server.nodes.map {|node| node.redis.values.count}
# 每个nodes上存储的key的数量
=> [2411, 2531, 2531, 2527] 
```

分配还是比较平均的，接下来看一下增加一个节点的命中率

```
@server.add_node '127.0.0.5'

count = 0
@server.nodes.each 
keys.each do |key|
  count+=1 if @server.execute 'get',key
end  
puts count
=> 8068
```

基本上跟预测的一致 原有节点数/新的节点数 的命中率。

注：依然是简单实现，现实情况需要考虑是否热迁移，旧的key是否删除等问题。

## 4. 开源实现

比较常见的是twitter的Twemproxy的和国产的豌豆荚的codis，前者是C实现后者是Go实现的，相对来说，前者提供了更多的配置项而后者则提供了更多的功能性。至于官方的redis cluster，至少目前真的不推荐。

其他：

- Netflix/dynomite
- facebook/mcrouter