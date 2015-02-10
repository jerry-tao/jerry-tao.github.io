---
layout: post
title: "ActiveSupport::Inflector"
date: 2015-1-27 11:47:06
categories: sources
image: /assets/images/post.jpg

---

inflector是active_support中的内容，用来处理应对不同需求的字符串。例如post -> posts，post -> Post post_controller->PostController等。

所有跟inflector有关的文件都可以在activesupport/lib/inflector.rb里找到，所以你可以单独使用这个模块。

{% highlight ruby %}

# in case active_support/inflector is required without the rest of active_support
require 'active_support/inflector/inflections' # 使用singleton模式定义了一个存储所有转换规则的空间。
require 'active_support/inflector/transliterate' # 把非ASCII字符转换为ASCII字符还有把字符串转换成适合URL的格式。
require 'active_support/inflector/methods' # 具体的转换方法。

require 'active_support/inflections' # 内置的转换规则。
require 'active_support/core_ext/string/inflections' # 扩展ruby的字符串。

{%endhighlight%}

先来看一下active_support/lib/inflector/inflections.rb里的几行关键代码：

{% highlight ruby %}

module ActiveSupport
  module Inflector
    extend self
    class Inflections
      @__instance__ = ThreadSafe::Cache.new #定义一个存储空间
      ...
      def initialize
        @plurals, @singulars, @uncountables, @humans, @acronyms, @acronym_regex = [], [], [], [], {}, /(?=a)b/
        #定义各个规则的存储空间
        # @plurals - 转换成复数的规则
        # @singulars - 转换成单数的规则
        # @uncountables - 不可数的单词
        # @humans - 人性化的规则 the_title => the title
        # @acronym_reges - 不标准的一些规则 例如 RESTFul
      end

    end

    def inflections(locale = :en)
      if block_given?
        yield Inflections.instance(locale)
      else
        Inflections.instance(locale)
      end
    end
    # 这个使用方法可以看一下
    # ActiveSupport::Inflector.inflections(:en) do |inflect|
    #   inflect.plural(/$/, 's') # 添加一条复数规则
    # end
  end
end

{%endhighlight%}

保存好之后在active_support/lib/inflections.rb里我们可以找到内置的规则，其中包括单复数不规则名词和不可数名词。

然后在active_support/lib/inflector/methods.rb里面定义了各种方法，随便看一个pluralize的方法：

{% highlight ruby %}

def pluralize(word, locale = :en)
  apply_inflections(word, inflections(locale).plurals)
end

def apply_inflections(word, rules)
  result = word.to_s.dup

  if word.empty? || inflections.uncountables.include?(result.downcase[/\b\w+\Z/])
    result
  else
    rules.each { |(rule, replacement)| break if result.sub!(rule, replacement) }
    result
  end
end
{% endhighlight %}

然后看一下active_support/lib/inflector/transliterate.rb，这里定义了两个方法：
transliterate和parameterize:

{% highlight ruby %}

transliterate('Ærøskøbing')
# => "AEroskobing"

"Donald E. Knuth".parameterize
# =>"donald-e-knuth"
{% endhighlight %}

最后，整理一下inflector里常用的方法：

{% highlight ruby %}
'post'.pluralize # 复数
# => "posts"
'post'.singularize # 单数
# => "post"
'post_controller'.camelize # 驼峰
# => "PostController"
'PostController'.underscore # 下划线
# => "post_controller"
'post_controller'.humanize # 可读 首字母大写
# => "Post controller"
'post_controller'.titleize
# => "Post Controller" # 标题适用
'Post'.tableize # 建表适用
# => "posts"
'post_controller'.classify # 类名适用
# => "PostController"
'post_controller'.dasherize # 替换成-
# => "post-controller"
'ActiveRecord::Base'.demodulize # 去模块化
# => "Base"
'ActiveRecord::Base'.deconstantize # 去常量化
# =>"ActiveRecord"
'Post'.foreign_key # 生成外键
# => "post_id"
class Post
end  
'Post'.constantize # 查找一个常量 不存在会报错
# =>Post
'Post'.safe_constantize # 同上 不存在nil
# =>Post
1.ordinalize # 把数字变成1st的模式
# =>'1st'
{% endhighlight %}
