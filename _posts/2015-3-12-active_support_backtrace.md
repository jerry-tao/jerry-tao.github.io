---
layout: post
title: "ActiveSupport BacktraceCleaner"
date: 2015-3-12 11:47:06
categories: sources
image: /assets/images/glasses.jpg

---

lol, 这一篇完全是为了凑数的。

在ruby中raise一个异常的同时会给这个异常一个异常栈，是一个字符串的数组，类似于 RuntimeError, line 3之类的信息。

例如

```ruby
[6] pry(main)> begin
[6] pry(main)*   raise RuntimeError
[6] pry(main)* rescue RuntimeError=>error
[6] pry(main)*   error.backtrace.size
[6] pry(main)* end
#=> 28 #这是在我这个环境里的数字 其中包括了pry 一直到rbenv的异常
```

这个苦逼的数字就是说，我们raise一个异常，backtrace里就会有N多调用信息。为了精简这些信息，就有了ActiveSupport::BacktraceCleaner(说白了就是个字符串数组过滤器)。

BacktraceCleaner提供两种过滤，一种是修改，一种是过滤掉。

```ruby
bc = ActiveSupport::BacktraceCleaner.new
bc.add_filter {|line| line.sub('/Users/jerry','~')} #把绝对路径换成相对路径
bc.add_silencer {|line| !line.index('rbenv').nil?} #把包含rbenv的过滤掉
#! 执行顺序是先filter然后silencer
bc.clean(error.backtrace) #上面例子里的error
```

下面的是rails默认的backtrace cleaner。

```ruby
require 'active_support/backtrace_cleaner'

module Rails
  class BacktraceCleaner < ActiveSupport::BacktraceCleaner
    APP_DIRS_PATTERN = /^\/?(app|config|lib|test)/
    RENDER_TEMPLATE_PATTERN = /:in `_render_template_\w*'/
    EMPTY_STRING = ''.freeze
    SLASH        = '/'.freeze
    DOT_SLASH    = './'.freeze

    def initialize
      super
      @root = "#{Rails.root}/".freeze
      add_filter { |line| line.sub(@root, EMPTY_STRING) }
      add_filter { |line| line.sub(RENDER_TEMPLATE_PATTERN, EMPTY_STRING) }
      add_filter { |line| line.sub(DOT_SLASH, SLASH) } # for tests

      add_gem_filters
      add_silencer { |line| line !~ APP_DIRS_PATTERN }
    end

    private
    def add_gem_filters
      gems_paths = (Gem.path | [Gem.default_dir]).map { |p| Regexp.escape(p) }
      return if gems_paths.empty?

      gems_regexp = %r{(#{gems_paths.join('|')})/gems/([^/]+)-([\w.]+)/(.*)}
      gems_result = '\2 (\3) \4'.freeze
      add_filter { |line| line.sub(gems_regexp, gems_result) }
    end
  end
end
```

再稍微延伸一点点，其实bc的源码一点都不复杂，所以自己绝对可以看明白，在以后有过滤一个数组的需求的时候可以考虑一下bc的实现方式。
