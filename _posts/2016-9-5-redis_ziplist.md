---
layout: post
title: "Redis Ziplist总结"
date: 2016-9-4 12:47:06
categories: Redis
image: /assets/images/old-1130743_1280.jpg
---

对redis的ziplist做个总结。

先说结论：***在合理配置下，ziplist可以大幅度减少redis的内存占用，并且几乎不影响性能***。

下面是正文。

ziplist其实是通过把短结构压缩在一起来达到节省内存的做法。

以我们常用的LIST结构为例：

实际上在Redis内部LIST的定义是一个双向链表：

```
typedef struct list {
    listNode *head;
    listNode *tail;
    unsigned long len;
} list;

typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;
```

对于一个长度为3的List来说，包含一个list对象和三个listNode对象。

对于每个node来说，存储了三个指针，prev，next和value，一个指针占4个字节。

其中value就是一个标准的STS对象。内存包含了长度，空余长度，真实字符串+\0

以一个节点'abc'来说，真实字节是3bytes，而一个节点需要额外占用21bytes（三个指针，两个int，一个\0）的内存，总占用就是3+21=24bytes。

而对于ziplist来说，采用的是类似数组的结构，结构如下：

```
<zlbytes><zltail><zllen><entry><entry><zlend>
```

其中entry可以简单分为三部分，前一个entry的长度，还有自己的encoding，自己的内容。

而对于前一个entry的长度来说，只要长度不超过255都只占一个bytes，如果超过255则都占5bytes（这就是为什么使用ziplist只适合短结构）

同样以'abc'为例，假设前面元素长度是5，则abc这个entry的总长度是3+2=5bytes。

上面的例子可以看到对内存使用的影响，然后看一下对性能的影响。

相对于链式结构来说，顺序结构本来应该有更快的访问速度的，但是因为这个顺序结构并不是标准的数组，无法通过下标来访问指定元素，所以查找的速度基本也是O(N)。

而相对删除插入等操作就更糟糕，因为还要涉及到重新分配以及移动内存等问题，所以基本在性能上对比link_list是没有优势的。

不过因为存储的内容不多，所以连续操作情况也还算可以接受，基本上影响不大。（在ziplist很短的情况下）

然后看一下配置

```
list-max-ziplist-entries 512 # 最大接受长度为512，超过此长度则转换为linked_list的存储模式。
list-max-ziplist-value 64 # 每个元素的大小，最大不超过64bytes，超过则转换为linked_list。
```

查看配置的例子

```
redis.lpush 'zip_demo',1
redis.debug(:object,'zip_demo')
# => "Value at:0x7f8714301a10 refcount:1 encoding:ziplist serializedlength:14 lru:13447321 lru_seconds_idle:11"

512.times do 
    redis.lpush 'zip_demo',1
end
# => "Value at:0x7f8714301a10 refcount:1 encoding:linkedlist serializedlength:1028 lru:13447378 lru_seconds_idle:3"

redis.lpush 'zip_demo1',1
redis.debug(:object,'zip_demo1')
# => "Value at:0x7f871434ab10 refcount:1 encoding:ziplist serializedlength:14 lru:13447427 lru_seconds_idle:1"
redis.lpush 'zip_demo1','a'*80
redis.debug(:object,'zip_demo1')
# => "Value at:0x7f871434ab10 refcount:1 encoding:linkedlist serializedlength:16 lru:13447449 lru_seconds_idle:4"

````
这里需要说明的是，如果一个list转换为了linkedlist，这个过程是不可逆的，即使你把长的node删掉，或者把list元素的个数降到配置以下，也不会再变为ziplist了。

最后做一些测试来决定配置（取决于具体机器以及使用情况）：

基本上list-max-ziplist-value这个值设定在64是没有问题的，可以适当放大，但是这个值最大不应该超过250，超过250有可能会带来更多的连锁内存反应。

```
$redis = Redis.new
def lpush(count)
  count.times do
    $redis.lpush('bench_redis',1)
  end
end
def  bench_redis
  Benchmark.bm do |x|
    x.report {lpush 1000}
    $redis.del 'bench_redis'
    x.report {lpush 5000}
    $redis.del 'bench_redis'
    x.report {lpush 10000}
    $redis.del 'bench_redis'
    x.report {lpush 50000}
    $redis.del 'bench_redis'
    x.report {lpush 99999}
  end  
  $redis.del 'bench_redis'
end

# 配置 list-max-ziplist-entries 1

       user     system      total        real
   0.070000   0.070000   0.140000 (  0.129996)
   0.300000   0.280000   0.580000 (  0.494019)
   0.600000   0.570000   1.170000 (  1.003400)
   2.870000   2.780000   5.650000 (  4.755737)
   5.690000   5.510000  11.200000 (  9.407786)

# 配置 list-max-ziplist-entries 100000

       user     system      total        real
   0.080000   0.070000   0.150000 (  0.133488)
   0.290000   0.290000   0.580000 (  0.490396)
   0.610000   0.560000   1.170000 (  1.001671)
   3.000000   2.770000   5.770000 (  4.878880)
   6.990000   5.980000  12.970000 ( 36.726020)
```

上面单一操作显示在5w左右的长度ziplist还可以保持住单一cli的写操作，但是这个测试里面还是有几点需要注意，一般来说我们不会只有写操作，相对其他操作而言写操作算是压力比较小的操作了。

所以这个值不要超过1w还是可以接受的，而且还可以根据具体使用情况来进一步缩短。

Reference:

- [Redis in Action](https://book.douban.com/subject/10597898/)
- [Redis设计与实现](https://book.douban.com/subject/25900156/)