---
layout: post
title: "Use Postgresql uuid in Rails"
date: 2015-2-4 11:47:06
categories: Rails
image: /assets/images/post.jpg
---

使用uuid来替换integer的id的好处有两个：

- 多个数据库的时候不会产生重复ID
- 避免对外泄漏数据（比如说如果是integer的订单系统，很容易通过自增来判断数据大小。）

因为postgresql原生就支持uuid，所以只需要在rails中开启一下特性即可。

新建一个migration文件（最好手工建立，确保时间戳在其他migration之前）：

{% highlight ruby %}
class AddUuidExtension < ActiveRecord::Migration
  def change
    enable_extension 'uuid-ossp'
  end
end

{%endhighlight%}

然后你可能需要手工修改以后的或者已经存在的migration，

{% highlight ruby %}
create_table :products, id: :uuid do |t| # 在建表时指定ID类型
  ...
  #在引用部分的修改。
  t.uuid :order_id
end

{%endhighlight%}

### One more thing
还有额外的一点，当你使用uuid之后，如果使用factory_girl来构建假数据可能出现的问题。

这个取决于你是否有验证unique，当你验证unique这个属性的同时使用里`build_stubbed`的时候，因为`factory_girl`默认生成假ID的方法是1000+x, 但是unique这个验证却是需要检查数据库的，所以会产生一个ERROR。

下面有一个快速的修复方法：

{% highlight ruby %}
#spec/supports/factory_girl.rb
require 'factory_girl_rails'

module FactoryGirl
  module Strategy
    class Stub
      private
      def next_id
        SecureRandom.uuid
      end
    end
  end
end

RSpec.configure do |config|
  config.include FactoryGirl::Syntax::Methods
end

{%endhighlight%}

还有不要忘记在你的`spec_helper`里面require supports下的文件:

`Dir[Rails.root.join('spec/support/**/*.rb')].each { |f| require f }`
