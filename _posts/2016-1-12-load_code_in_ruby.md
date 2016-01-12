---
layout: post
title: "Ruby加载代码的二三事"
date: 2016-1-12 14:47:06
categories: ruby
image: /assets/images/old-1130743_1280.jpg

---

## 1. Ruby的require、load和autoload

除了写很简单的程序之外，大多数情况都不会只使用一个文件来存储所有代码，所以了解一下文件是如何相互组织的有助于理解整体的关系。

假设我们在一个文件夹内有以下两个文件：

```
# codes/person.rb
class Person
  def greet
    puts "Hello."
  end
end

# codes/main.rb
person = Person.new
person.greet
```

在main.rb中我们并不能直接使用person.rb中的代码，需要先把person.rb中的代码require进来。

```
# codes/main.rb

require './person'
```

使用require的时候并不需要写后缀，默认既是rb，也可以指定其他的后缀例如socket.so等。

当require成功返回true，已经require的文件返回false，找不到抛出LoadError。


最基本的require的语法是require加路径(相对绝对均可)，但是有个问题，当使用相对路径时，默认的当前路径并不是固定不变的，如果想使用绝对路径，又会由于不同人的系统不同人的环境导致很多问题。


例如如下文件夹结构：

```
├── main.rb
├── person.rb
└── scripts
    └── run.rb

# run.rb
require '../main'
```

当我们在scripts里执行run.rb的时候，run会require main，在main里又会require person，但是此时的当前路径是scripts/，所以会找不到person.rb。

所以接下来我们需要了解一下$LOADED\_FEATURES(aka $"),$LOAD\_PATH(aka $:)。

$LOADED\_FEATURES里保存了已经require过的文件，$LOAD\_PATH里保存了查找文件的路径，两者都是数组结构。

以`require 'benchmark'`为例，在不考虑更复杂的（例如包含了gem或者rails的情况下，下面还会继续介绍），典型的查找流程是这样，现在$LOADED\_FEATURES里查找是否已经require过这个文件，如果已经require过，则返回false，如果没有，则会在$LOAD\_PATH内依次查找这个文件。

继续上面的例子，改写如下：

```
# codes/main.rb
require 'person'

# codes/scripts/run.rb
require 'main'
```

然后在运行的时候`ruby -I. scripts/run.rb ` (-I.代表把当前路径添加到$LOAD_PATH内，你也可以写全路径。)

你可以在大多数的gem源码中的gemspec里看到这句话，这句话的意思就是在当打包gem的时候把lib添加到$LOAD\_PATH内。

```
lib = File.expand_path('../lib', __FILE__)
$LOAD_PATH.unshift(lib) unless $LOAD_PATH.include?(lib)
```

以上介绍的是ruby里最基本的require，接下来还会有其他的一些信息，不过我们还是先了解一下load和autoload。

load在功能和使用上跟require都很相似，主要有以下四点不一样：

- load的参数必须是全文件名 `load 'person.rb'`
- load的文件不会添加进$LOAD\_FEATURES内
- 同一文件可以被load多次
- load可以把本地变量暴露给被load的文件

其中前三点不需要说明，第四点还是需要说一下。

假设我们有以下文件：

```
# say.rb
puts name

# name.rb
name = "Rails"

# require './say' # 会报错，找不到变量name

load './say.rb' # 可以正常执行，load 可以把本地变量暴露给load进来的文件

# load './say.rb', wrap: true 会报错，这个会使用一个匿名的module来执行load进来的代码

```

autoload虽然起名叫做load，但实际上的工作机制是require，用来实现一种懒加载的机制。

当我们使用autoload的时候，仅仅是注册了一个常量，真正的代码还没有被require进来，当我们第一次使用这个常量时，则会去require注册的文件。还是以一开始的代码为例：

```
# codes/main.rb
autoload :Person, './person.rb'
puts "LOADED_FEATURES: #{$".select {|item| item.index('person')}}" 
p = Person.new
puts "LOADED_FEATURES: #{$".select {|item| item.index('person')}}"
# => LOADED_FEATURES: []
# => LOADED_FEATURES: ["/path/demo/person.rb"]
```


## 2. Gem的require和bundler

实际上上面的require说明的情况仅仅适用于最基本的情况，require是Kernel.require 方法，而实际上我们使用的往往不是这个方法。

```
# 打开irb输入
method(:require).source_location
=> ["/usr/local/var/rbenv/versions/2.2.2/lib/ruby/2.2.0/rubygems/core_ext/kernel_require.rb", 38]
```

因为rubygems已经重写了require方法，并且rubygems现在以及随ruby语言一起分发，所以在我们没有进一步重写过require我们使用的都是这个方法。

所以在使用这个require的时候，当$LOAD\_PATH内找不到的这个文件，就会使用Gem.find\_files 'file' 来查找 如果有结果 就会require对应的文件。

举例 我们有个gem y , 里面有个y.rb

当我们require 'y'的时候，因为LOAD\_PATH内没有 rubygems就会通过Gem.find\_files 'y' 在gems的require\_paths内查找，如果有结果 就会require对应的文件。

当gem y内的y.rb被require的时候，也会自动把在gemspec里设置的require\_paths加到$LOAD\_PATH

可以在Gem的gemspec里找到如下的代码：


```
Gem::Specification.new do |spec|
  ...
  spec.require_paths = ["lib"]
end
```

使用bundler主要是下面这个方法：

```
require 'bundler/setup'
Bundler.require(*Rails.groups)
```

这里不继续深入了，最简单的理解可以理解为bundler替你依次require了你需要的版本的Gem和你指定require的文件（如果有指定）。