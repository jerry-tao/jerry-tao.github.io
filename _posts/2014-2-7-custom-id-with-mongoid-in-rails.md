---
layout: post
title:  "自定义mongoid的model id"
date:   2014-02-7 11:47:06
categories: Ruby Gems
image: /assets/images/post.jpg
---
如果在rails使用mongoid，在返回json的时候会有个小问题，就是id格式会如下：


{% highlight javascript %}
  {
    "code": 200,
    "data": [
        {
            "id": {
                "$oid": "52f472cc6a65721a2f000000"
            },
            "name": "abc"
        }
    ]
  }
{% endhighlight %}

过去是用如下代码修改：

{% highlight ruby %}
Moped::BSON::ObjectId.class_eval do
  def to_json(*args)
    to_s.to_json
  end
  #Jbuilder使用的to json策略
  def as_json(*args)
    to_s.as_json
  end
end
{% endhighlight %}

如果用的是git上最新的mongoid代码，BSON是在mongoid自己的extensions里的，所以修改如下：

{% highlight ruby %}
BSON::ObjectId.class_eval do
  def to_json(*args)
    to_s.to_json
  end
  #Jbuilder使用的to json策略
  def as_json(*args)
    to_s.as_json
  end
end
{% endhighlight %}

看下结果：

{% highlight javascript %}
  {
    "code": 200,
    "data": [
        {
            "id": "52f472cc6a65721a2f000000",
            "name": "abc"
        }
    ]
  }
{% endhighlight %}
