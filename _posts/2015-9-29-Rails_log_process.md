---
layout: post
title: "Rails Log Process"
date: 2015-9-29 14:47:06
categories: rails
image: /assets/images/chemistry-740453_1280.jpg

---



最近研究了一下整理Rails的日志，简单整理下。

使用工具包括：

- logstash
- lograge
- elasticsearch

## 0. 准备

如果没有修改过任何配置，rails的日志大概是这样的

```
Started POST "/users" for ::1 at 2015-09-28 15:46:22 +0800
Processing by Devise::RegistrationsController#create as HTML
  Parameters: {"utf8"=>"✓", "authenticity_token"=>"m/JixAv0ZmXTRLn1mmFGKNF/1MQgb5iXqPThmc3S23B7eiYfCmzwDXALr5yrw9v0dUWdkGVEkGzt8Qa2Q9vlFQ==", "user"=>{"email"=>"user@example.com", "password"=>"[FILTERED]", "password_confirmation"=>"[FILTERED]"}, "commit"=>"Sign up"}
   (0.1ms)  begin transaction
  User Exists (0.1ms)  SELECT  1 AS one FROM "users" WHERE "users"."email" = 'user@example.com' LIMIT 1
   (0.1ms)  rollback transaction
  Rendered /usr/local/var/rbenv/versions/2.2.2/lib/ruby/gems/2.2.0/gems/devise-3.4.1/app/views/devise/shared/_links.html.erb (1.6ms)
  Rendered /usr/local/var/rbenv/versions/2.2.2/lib/ruby/gems/2.2.0/gems/devise-3.4.1/app/views/devise/registrations/new.html.erb within layouts/application (11.1ms)
Completed 200 OK in 148ms (Views: 51.5ms | ActiveRecord: 0.3ms)
```

这个时候使用的logger是ActiveSupport里提供的默认logger。

Rails log机制是通过pub/sub来实现的，比如ActiveRecord::LogSubscriber，还是很方便重新改写的。

（关于Notification可以翻看以前的一篇文章。[ActiveSupport Notifications](http://blog.jerry-tao.com/sources/2015/02/23/active_support_notifications.html)）


了解这些基础之后让我们假设一个我们需要的数据格式（伪model）：

```
class SQL
  belongs_to :request
  attributes :uuid # 用来跟request关联的id
  attributes :session_id # 用来跟request关联的id
  attributes :duration # 时间
  attributes :name # 存储对应的模型名称
  attributes :message # 存储对应的SQL语句
  attributes :type # 区别其他两种类型
end

class ExceptionStack
  belongs_to :request
  attributes :uuid # 用来跟request关联的id
  attributes :session_id # 用来跟request关联的id
  attributes :message # 错误信息
  attributes :error # 错误类
  attributes :backtrace # 异常栈
  attributes :status # 返回的状态
  attributes :handler # 处理的方法
  attributes :type # 区别其他两种类型
end

class Request
  attributes :uuid # 每一次请求的唯一标志
  attributes :session_id # 一次session的id
  attributes :method # HTTP Verb
  attributes :path # 请求地址
  attributes :format # 请求格式
  attributes :controller # 对应controller
  attributes :action # 对应action
  attributes :status # 返回状态
  attributes :duration # 时间
  attributes :view # 渲染页面时间
  attributes :db # 处理数据库时间
  attributes :type # 区别其他两种类型
  attributes :host # 主机地址
  attributes :remote_ip # 请求IP
  attributes :origin # 来源请求的页面
  attributes :user_agent # 用户浏览器信息
  attributes :@timestamp # 时间
  attributes :message # 请求概述
end
```

## 1. 格式化Rails日志


### 1.1 Request日志

先加lograge和logstash-event到Gemfile

```
gem 'lograge'
gem 'logstash-event'
```

然后在application.rb里先简单配置一下：

```
config.lograge.enabled = true
config.lograge.formatter = Lograge::Formatters::Logstash.new
config.colorize_logging = false # 关闭彩色显示，会产生很多不适合阅读的字符。
```

这个时候输出的结构大概是：

```
{
    "method": "POST",
    "path": "demo/auth/sessions",
    "format": "json",
    "controller": "demo/auth/sessions",
    "action": "create",
    "status": 401,
    "duration": 27.2,
    "view": 0.22,
    "db": 11.32,
    "@timestamp": "2015-09-29T10:53:14.766Z",
    "@version": "1",
    "message": "[401] POST demo/auth/sessions (demo/auth/sessions#create)"
}
```

距离我们要的数据还缺很多信息，根据lograge的文档，需要先在application把需要的数据加进来

```
# application_controller.rb
def append_info_to_payload(payload)
  super
  payload[:uuid] =  request.uuid
  payload[:session_id] = request.cookie_jar['_session_id']
  payload[:host] = request.host
  payload[:remote_ip] = request.remote_ip
  payload[:origin] = request.headers['HTTP_ORIGIN']+request.headers['ORIGINAL_FULLPATH']
  payload[:user_agent] = request.headers['HTTP_USER_AGENT']
end
```

然后就可以在application.rb自定义一下这些内容了：

```
#application.rb
config.lograge.custom_options = lambda do |event|
  {
    uuid: event.payload[:uuid],
    session_id: event.payload[:session_id],
    type: 'request',
    host: event.payload[:host],
    remote_ip: event.payload[:remote_ip],
    origin: event.payload[:origin],
    user_agent: event.payload[:user_agent]
  }
end
```

这样请求的格式就正确了。


### 1.2 SQL日志

SQL的日志默认是通过ActiveRecord::LogSubscriber来记录的，这个会包含格式化SQL语句等功能，但是我们想把他转换成json，适合通过logstash收集进elasticsearch。

我用了一种很取巧的手段：

```
#config/initializers/sql_log.rb
# 先把默认的subscriber去掉。
Lograge.module_eval do
  ActiveSupport::LogSubscriber.log_subscribers.each do |subscriber|
    case subscriber
    when ActiveRecord::LogSubscriber
      unsubscribe(:active_record, subscriber)
    end
  end
end

# 自己实现一个subscriber
module SQLLog
  class LogSubscriber < ActiveSupport::LogSubscriber
    IGNORE_PAYLOAD_NAMES = ["SCHEMA", "EXPLAIN"]

    def self.runtime=(value)
      ActiveRecord::RuntimeRegistry.sql_runtime = value
    end

    def self.runtime
      ActiveRecord::RuntimeRegistry.sql_runtime ||= 0
    end

    def self.reset_runtime
      rt, self.runtime = runtime, 0
      rt
    end

    def initialize
      super
    end

    def render_bind(column, value)
      if column
        if column.binary?
          # This specifically deals with the PG adapter that casts bytea columns into a Hash.
          value = value[:value] if value.is_a?(Hash)
          value = value ? "<#{value.bytesize} bytes of binary data>" : "<NULL binary data>"
        end

        [column.name, value]
      else
        [nil, value]
      end
    end

    def sql(event)
      self.class.runtime += event.duration
      return unless logger.debug?

      payload = event.payload

      return if IGNORE_PAYLOAD_NAMES.include?(payload[:name])

      unless (payload[:binds] || []).empty?
        binds = "  " + payload[:binds].map { |col, v|
          render_bind(col, v)
        }.inspect
      end
      ids = Thread.current[:log_uuid_session_id] || Array.new(2)
      log = {
        name: payload[:name],
        duration: event.duration.round(1),
        binds: binds,
        message: payload[:sql],
        type: 'sql',
        uuid: ids[0],
        session_id: ids[1] }.to_json
      debug log
    end


    def logger
      ActiveRecord::Base.logger
    end
  end
end

# attach 到active_record上
SQLLog::LogSubscriber.attach_to :active_record

```

因为在这里面我想存储一下关联的request和session，所以才有了`ids = Thread.current[:log_uuid_session_id] || Array.new(2)`。

如果不需要直接去掉就好，如果需要还需要在application_controller里记录一下：

```
# application_controller
before_filter :set_uuid_session_id

private

def set_uuid_session_id
  Thread.current[:log_uuid_session_id] = [request.uuid,request.cookie_jar['_session_id']]
end
```

这样可以很方便的看到一次请求产生了几条db请求。


### 1.3 异常日志

首先，rescue所有的异常是很有争议的做法，不过我还是想对用户尽量屏蔽这些异常情况。

```
# application_controller.rb
rescue_from Exception, with: :generic_exception # 可能还有很多其他的异常rescue

def generic_exception(error)
  ids = Thread.current[:log_uuid_session_id]
  if error.is_a?(Class) && error <= Exception # 这里是因为有些时候会捕捉到例如RuntimeError这种情况。
    error_class = error.name
    error_message = error.name
    backtrace = []
  else
    error_class = error.class.name
    error_message = error.message
    backtrace = error.backtrace
  end
  log = { error: error_class, backtrace: backtrace, type: 'exception', status: 500, message: error_message, handler: 'generic_exception', uuid: ids[0],
          session_id: ids[1] }
  Rails.logger.error log.to_json
  render nothing: true, status: 500
end
```

通过上面的处理，适合输入到logstash 的json格式日志算是准备好了。

## 2. 使用logstash收集log

关于logstash，还有多个rails server的情况，可以去参考logstash的文档，我就简单贴一下基本的配置：

```
input {
    stdin {}
  file {
    type=> 'application'
    path=>'~/Workspaces/demo/log/development.log'
    codec=>'json'
  }
}



output {
  stdout {}
  elasticsearch {
    cluster=> 'logstash'
    protocol => http
  }
}

```

不适合生产环境，如果使用多个集群还需要考虑使用redis来做中转，根据具体情况具体修改吧。

然后测试下：

```
elasticsearch -d
logstash -f sample.conf
rails s
```

随便访问几个网址，然后看一下ES里的数据：

```
curl -XGET "http://localhost:9200/logstash-2015.09.29/_search" -d'
{
  "sort": [
    {
      "@timestamp": {
        "order": "desc"
      }
    }
  ]
}'
```

默认情况下会使用logstash-日期 这种格式的index。

至于如何显示如何处理，就是另一个topic了。

有很多细节地方还可以优化，例如把异常的log也使用notification，把自己写的notification集成进lograge等，不过大体的思路还是这样的。
