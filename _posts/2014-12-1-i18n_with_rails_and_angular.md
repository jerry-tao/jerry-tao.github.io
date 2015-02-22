---
layout: post
title: "I18n with Rails and Angular"
date: 2014-12-1 11:47:06
categories: Rails And Angular
image: /assets/images/post.jpg

---

这里简单记录一下rails和angular一起完成i18n的流程。

# Part 1: I18n with Rails
### 写locales文件
locale文件默认存在于config/locales/

可修改，定义位置在application配置里

{% highlight ruby %}

config.i18n.default_locale=:en #设置默认的语言

#config/locales/

#config/locales/page.yml

#config/locales/message.yml

{% endhighlight %}


locale默认是yml格式，比如下面就是定义英文的key

{% highlight ruby %}

en:
  hello: Hello Wrold

{% endhighlight %}

### 设置locale

这里的核心就是在后台要获取到需要使用的语言环境。

最简单的办法：

在application_controller里面添加：

{% highlight ruby %}

before_action :set_locale

def set_locale
  I18n.locale = params[:locale] || I18n.default_locale
end

{% endhighlight %}

这样例如下面的请求就会使用de.yml里的key

`http://localhost:3000?locale=de`

### 翻译内容
两个地方：

  - 返回的message里
  - view里面直接翻译

其实都一样：

{% highlight ruby %}

#在controller里
redern json: {hello: t(:hello)}
#在view里
<%=t(:hello)%>

{% endhighlight %}


这两种翻译都会生成Hello World

# Part 2: localeapp with Rails

在localapp.com新建一个project。

{% highlight ruby %}

#然后添加localeapp的gem。
`gem 'localeapp'`
#根据你的key安装一下
`bundle exec localeapp install <API_KEY>`

{% endhighlight %}

{% highlight ruby %}
#config/initializers/localeapp.rb
require 'localeapp/rails'
Localeapp.configure do |config|
  config.api_key = 'YOUR_API_KEY_HERE'
  config.sending_environments = []
  config.polling_environments = []
  config.reloading_environments = []
end
{% endhighlight %}

可以在项目内完成en.yml 然后推送至服务器交给其他人完成其他语言的翻译。

# Part 3: angular-translate with rails

在你的bower.json里添加`"angular-translate":"latest"`

在你的html里引用这个angular-translate.js

在你的angular里配置一下angular-translate

{% highlight coffee %}

var app = angular.module('myApp',['pascalprecht.translate'])

angular.module 'app'
.config ($translateProvider)->
  $translateProvider.preferredLanguage('en')
  $translateProvider.useUrlLoader('YOUR_DOMAIN:YOUR_PORT/locales')

{% endhighlight %}

在你的后台写一个action去响应这个请求 render json


{% highlight ruby %}
def locales
  locale_folder = File.join(Rails.root, 'config/locales')
  @locale_json = {}

  Dir[File.join(locale_folder, '*.yml')].sort.each { |locale|
    locale_yml = YAML::load(IO.read(locale))
    lang = params['lang'].nil? ? I18n.default_locale.to_s : params['lang']
    locale_hash = locale_yml.to_hash[lang]
    @locale_json.merge! locale_hash unless locale_hash.nil?
  }
  render json: @locale_json
end

{% endhighlight %}

在你要翻译的地方 可以使用以下三种翻译方式

{% highlight coffee %}

{% raw %}{{'hello' | translate}} # filter {% endraw %}

<h4 translate='page.title'> # directive

$translate(['page.title']).then (translations) ->
    $scope.page_title = translations["page.title"] # provider

{% endhighlight %}

然后在页面上直接使用 `{% raw %}{{page_title}}{% endraw %}`即可

注： `<h4 translate='page.title'>`这种使用方式目前还是有bug的 尽量使用在不复杂的页面环境下。

切换不同的语言使用`$translate.uses('de_DE')`。


## References

- rails-i18n: [http://guides.rubyonrails.org/i18n.html](http://guides.rubyonrails.org/i18n.html)

- localeapp: [https://github.com/Locale/localeapp](https://github.com/Locale/localeapp)

- angular-transllate: [https://github.com/angular-translate/angular-translate](https://github.com/angular-translate/angular-translate)

<!--
## Part 4 I18n的一些细节
## 不支持太详细的设定 可以使用globalize3设定
## 如果想翻译数据还是很麻烦的 需要使用globalize3来翻译数据部分
## i18n gem包含两部分 public 和 backend backend可以随便换
public api部分包括
 translate localize
 t   l
 load_path locale default_locale exception_handler backend
 -->
