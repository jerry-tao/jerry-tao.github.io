---
layout: post
title: "实现一个简单的Gateway"
date: 2017-3-8 20:47:06
categories: Fun
image: /assets/images/stopwatch-2061845_1280.jpg
---

Gateway是复杂系统中常见的基础组成部分，可以理解为一个包含简单逻辑的反向代理，例如https://github.com/hellofresh/janus，一般大体位置如图：

![gateway](/assets/images/gateway.png)

这其中可能还包含权限认证以及服务管理，api version，api scope的权限等。接下来基于mruby介绍下各部分的简单实现。


## 1. 认证部分

在我们的系统内存在一个认证服务，认证通过之后则会返回给前台一个JWT token，之后通过JWT token来实现权限认证。

之所以选择JWT是因为JWT是一个自包含信息的结构，在简单场景下不需要再次与数据库或者缓存沟通就可以完成验证。

```
# 认证服务
if user = User.resolve(username,password)
  payload = {auth: true, uid: user.id, admin: user.admin}
  return  {auth: true, token: JWT.encode(payload,JWT_KEY,'HS256')}
else
  reuturn {auth: false}
end
```

在这里的JWT_KEY是保证安全的关键，一定要妥善保存好，并且JWT有一些预留字段，例如过期时间等，也非常适合实现更复杂的认证模式。

mruby的JWT解析选择的是https://github.com/prevs-io/mruby-jwt (这个只支持HS256的签名,而且目测作者也不会更新了）。

```
# 验证部分
# Nginx Config
mruby_access_handler access_control.rb;

# access_control.rb
begin
    request = Nginx::Request.new
    key =  request.headers_in["Authorization"]
    payload = JWT.decode(key,JWT_KEY,"HS512")[0]
    if request.method!="OPTIONS" && !payload['auth']
      Nginx::Request.new.headers_out["access-control-allow-origin"] = "*"
      Nginx.return Nginx::HTTP_UNAUTHORIZED
    else
      request.headers_in["UID"] = payload['uid']
    end
rescue 
    Nginx::Request.new.headers_out["access-control-allow-origin"] = "*"
    Nginx.return Nginx::HTTP_UNAUTHORIZED
end 
```

允许options请求进入用于跨域，并且如果验证通过，把uid加入到header避免后面服务进行二次JWT解码。

对于认证部分的几个注意细节，首先JWT完全是明文存储的，简单的base64解码就可以看到信息，千万不要存储任何敏感信息在里面，其次是虽然只解析JWT来判断验证是否通过是很理想的情况，但实际上也仅仅适用于比较简单的模型，如果需要做更复杂的权限管理还是要不可避免的去与数据库或者缓存沟通。

## 2. 服务管理

简单的说，就是我们会把所有服务的信息都存储起来，然后通过对应的数据去查找。

比如我们需要把所有post.example.com的请求都派发给后台的post_service监听的9999端口，我们可以在redis里存储一个hash

service_porxy = {
    'post.example.com': '127.0.0.1:9999'
}

同时在数据库中可能会有更详细的服务信息，是否允许跨域，需要什么权限，不同版本的动态切换，以及一定程度的负载均衡。

然后通过manage_service，来对外暴露service管理接口，监听vsc的webhook的来对service进行动态部署。

简化版的通过二级域名来选择后端服务接口的大概代码如下：

```
# Nginx Config

mruby_set $backend proxy.rb;
proxy_pass http://$backend;

# proxy.rb

redis = Redis.new host,port
r = Nginx::Request.new
redis.hget 'service_proxy', r.var.http_host
```

关于服务管理的一些细节，实际上这部分是把两个部分放在一起说了。

一部分是manage_service，功能包括：维护服务信息，以及动态部署服务。

这里面可以选用docker，同时在每个服务内写一个config.json文件，在自动部署的同时维护service信息，可以做到全程自动化。

还有一部分是用来根据service信息来选择动态代理的gateway，这里面可以做更多实质性的工作，例如不同版本的动态匹配，多个请求合并，移动端和web的自动适配，以及一定程度的负载均衡等等，。


## 3. 总结

只是用mruby来实现了一个比较简单的gateway结构的各个部分，如果要投入生产还需要做更多的细节处理，mruby在这里的作用跟lua基本是一致的。上面的例子是使用go实现的，其他备选还有lua+nginx（openresty）。
