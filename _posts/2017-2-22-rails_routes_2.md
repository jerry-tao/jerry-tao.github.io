---
layout: post
title: "Rails Routes 源码(2)"
date: 2017-2-22 20:47:06
categories: Rails
image: /assets/images/stopwatch-2061845_1280.jpg
---

这篇文章会详细的介绍定义路由的细节。

## 1. match细节

前面说的match只是个缩略的伪代码，实际上真正的match也只是个处理参数继续传递给map_match，并进一步传递给Mapping#add_route方法，

在这一步中重点的有这样几步：

```
def add_route(options)
  ast = Journey::Parser.parse path
  mapping = Mapping.build(@scope, @set, ast, controller, default_action, to, via, formatted, options_constraints, anchor, options)
  @set.add_route(mapping, ast, as, anchor)
end
```

第一步用来为当前路径生成AST（在查找的时候会详细说），第二部则是构建一个Mapping对象作为参数传递，第三步则是返回已经处理好的结果,传递给RouteSet#add_route。

```
route = @set.add_route(name, mapping)
```

然后继续添加给对应的Journey::Routes#add_route:

```
def add_route(name, mapping)
  route = mapping.make_route name, routes.length
  routes << route
  partition_route(route)
  clear_cache!
  route
end
```

make_route方法是来自于Mapping#make_route，继续处理了参数并生成Journey::Route对象

```
def make_route(name, precedence)
  route = Journey::Route.new(name,
                    application,
                    path,
                    conditions,
                    required_defaults,
                    defaults,
                    request_method,
                    precedence,
                    @internal)

  route
end
```

可以理解为，所有的路由规则最终都是以Journey::Route对象保存在Journey::Routes中的。在介绍匹配的时候会详细介绍。


## 2. 不同定义路由的实现

已经介绍过match的实现细节了，接下来看一下常见的其他路由实现。

```
# Basic
match 'home'=>'home#index'

# HTTP
get 'home'=>'home#index'
post 'home'=>'home#create'

# Scoping
scope path: "/admin" do
  resources :posts
end
scope module: "admin" do
  resources :posts
end

# Resources

resources :posts

# Concern

concern :commentable do 
  resources :comments
end

resources :posts do 
  member do 
    concern :commentable
  end
end

```

### HTTP

HTTP这种比较简单，基本就是变形的match，源码位于mapper内的HTTPHelpers内，以get为例，基本就是把get作为参数继续传递给match。

```
def get(*args, &block)
    map_method(:get, args, &block)
end

def map_method(method, args, &block)
    options = args.extract_options!
    options[:via] = method
    match(*args, options, &block)
    self
end

```

### Scoping

**额外的说下，这个部分是我觉得最有意思的部分了。**

scoping的作用是为scoping内的block提供一些默认的参数，例如path，module之类。

scoping的实现是通过设置了一个scope的树在内部通过切换不同的树节点来执行不同的block。伪代码基本如下：

```
class Mapping
  def initilize
    @scope = Scope.new
  end

  def scope(options,&block)
    @scope = @scope.new options
    block.call
    @scope = @scope.parent
  end
end
```

然后其他的方法诸如match会根据当前@scope来设置一些通用的参数。

除了scope这个方法之外，还提供了4个helper方法，用于快速设置scope，跟get对于match来说差不多，只是设置一些默认的scope方法。

- controller
- namespace
- constraints
- defaults


### Resources

简单地说，resources可以理解为如下代码：

```
def resources(resources)
    get resources,to: "#{resources.to_s}#index"
    post resources,to: "#{resources.to_s}#create"
    get "#{resources}/new",to: "#{resources.to_s}#new"
    get "#{resources}/:id/edit",to: "#{resources.to_s}#edit"
    get "#{resources}/:id",to: "#{resources.to_s}#show"
    put "#{resources}/:id",to: "#{resources.to_s}#update"
    delete "#{resources}/:id",to: "#{resources.to_s}#destroy"
end
```

内部的实现包括collection和member其实都是通过切换不同的scope来实现的。

### Mount

mount实际上挂的是一个rack app，把匹配到某个路径的请求全部调用对应的app的call方法：

```
match(path, options.merge(to: app, anchor: false, format: false))
```

在匹配的时候还会简单介绍下。


### Concern

concern的作用是提供一段共享的routes，如下例子所示：

```
concern :commentable do 
  resources :comments
end

resources :posts do 
  member do 
    concern :commentable
  end
end
```

concern一共提供了两个方法，一个用于定义，定义基本上就是把一个block存进当前mapping的@concerns，调用的时候则是把自己作为参数执行。伪代码如下：

```
def concern(name,&block)
  @concerns[name] = lambda{|mapper| mapper.instance_exec &block }
end

def concern(name)
  @concerns[name].call(self)
end
```