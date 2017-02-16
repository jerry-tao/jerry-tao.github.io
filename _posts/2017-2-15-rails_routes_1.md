---
layout: post
title: "Rails Routes 源码(1)"
date: 2017-2-15 12:47:06
categories: Rails
image: /assets/images/old-1130743_1280.jpg
---


Rails的routes的所有源码所在位置都在：

ActionPack => ActionDispatch

ActionDispatch这个模块就是负责用来处理HTTP相关以及跟rack一起使用的模块，可以分为下面四个部分：

- HTTP支持
- Rack Middlewares
- 定义路由部分
- 匹配路由部分

这篇文章先从整体简单介绍路由从定义到匹配的流程。

## 1. 定义路由

我们写路由的时候基本都是写在config/routes.rb里的，最简单的大概如下：

```
Rails.application.routes.draw do
  match 'posts', to: 'posts#index', via: :get
end
```

这里的routes是一个RouteSet的实例，在初始化RouteSet的时候就会初始化几个变量：

```
def initialize
  @set = Journey::Routes.new
  @router = Journey::Router.new @set
end
```

直接看draw的方法大概可以抽象如下：

```
def draw(block)
  Mapper.new(self).instance_exec(block)
end
```

把整个block交给mapper去执行，在mapper里的方法大概如下：

```
class Mapper
  def initialize(set)
    @set = set
  end

  def match(*argv) 
    @set.add_route(Mapping.build(argv))
  end  
end
```

在这里把路由都统一处理成mapping对象插入回RouteSet，

```
def add_route(mapping)
  @set.add_route(mapping)
end
```

在Journey::Routes中的add_route方法大概如下：

```
def add_route(mapping)
  @routes << Journey::Route.new(mapping)
end
```

至此，定义路由部分结束。

## 2. 匹配路由

上面已经说过，ActionDispatch还负责跟rack集成，这里面真正的rack app就是这个Rails.application，这里面可能中间还有很多其他的Rack middleware，包括在ActionDispatch中定义的，真正进入到匹配路由的依然是在RouteSet里的call方法。

```
def call(env)
  req = make_request(env)
  @router.serve(req)
end
```

上面也说过，这个router就是Journey::Router，

```
def serve(req)
  find_routes(req).each do |match, parameters, route|
    status, headers, body = route.app.serve(req)
    return [status,headers,body]
  end
end
```

上面的route就是一个Journey::Route对象，这里面的app就是在生成路由的时候指定的一个Dispatcher对象。

Dispatcher的serve就相对简单了，通过路由找到对应的controller和action就直接返回action的执行结果了：

```
def serve(req)
  params     = req.path_parameters
  controller = controller req
  res        = controller.make_response! req
  dispatch(controller, params[:action], req, res)
end
```

至此，整个路由从定义到匹配的流程就都已经完成了，接下来会更详细的介绍各个部分的具体内容。

整理版伪代码：

```
class RouteSet
  def initialize
    @set = Journey::Routes.new
    @router = Journey::Router.new @set
  end

  def draw(&block)
    Mapper.new(self).instance_exec(block)
  end

  def add_route(route)
    @set.add_route(route)
  end
  
  def call(env)
    req = make_request(env)
    @router.serve(req)
  end
end

class Dispatcher
  def serve(req)
    params     = req.path_parameters
    controller = controller req
    res        = controller.make_response! req
    dispatch(controller, params[:action], req, res)
  end
end

class Mapper
  def initialize(set)
    @set = set
  end

  def match(*argv) 
    @set.add_route(Mapping.new(argv))
  end  
end

class Mapping
  attr_accessor :app
  def new(*argv)
    normlize(argv)
    @app = Dispatcher.new
  end
end

class Journey::Router
  def serve(req)
    find_routes(req).each do |match, parameters, route|
      status, headers, body = route.app.serve(req)
      return [status,headers,body]
    end
  end
end

class Journey::Routes
  def initialize
    @routes = []
  end
  def add_route(route)
    @routes << Journey::Route.new(route)
  end
end

class Journey::Route
end

```