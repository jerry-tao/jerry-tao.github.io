---
layout: post
title: "Rails常用Gem"
date: 2015-2-9 11:47:06
categories: Rails
image: /assets/images/post.jpg
---

整理的只是我用到的，有一部分是rails项目自带的，我简单说一下作用。

查找gems的地方：

- [github](https://github.com/)
- [rubygems](https://rubygems.org/)
- [rubytool](https://www.ruby-toolbox.com/)


## Rails基础类

- [spring](https://github.com/spring/spring)：加速环境 4.x之后加入rails默认gem。

还有[spring-commands-rspec](https://github.com/jonleighton/spring-commands-rspec)

- [jbuilder]: 个人感觉比grape好用，不过看习惯，生成json的view。

## 测试类

- [rspec-rails]: 真的不用介绍。

- [factory_girl_rails]: 测试假数据就靠这个了。

- [database_cleaner]: 每次测试结束后自动帮你删除内容。

- [forgery]: 生成假信息的。

- [capybara](https://github.com/jnicklas/capybara): acceptance test。

- [simplecov]: 测试覆盖率。

## 数据库类

- [activerecord-session_store](https://github.com/rails/activerecord-session_store): 把session保存在数据库。

- [seedbank]: 默认的seed增强方案。

- [paranoia]: 软删除数据，不真正删除，而是加deleted_at。

- [annotate]: 自动注释model，spec/models

## 功能类

- [devise]: 大名鼎鼎的devise，不用解释了吧。

devise衍生出了很多gem，不一一描述。

- [omniauth]: Login with Google/Facebook/twitter....

- [devise_token_auth]: 如果你使用rails来做api，恰好使用angular来做前端，不要错过。

非angular也可以使用。

- [cancancan]: 权限管理，从cancan一路走来。

- [pundit]: 另一个权限管理，与cancancan齐名。

一般的建议是rails 4+ 使用pundit，rails 3.x使用cancancan。

- [active_model_serializers]:　另一种render json的方案，但是整体感觉没有jbuilder灵活

- [aasm]: 状态机，备选的还有state_machine。

- [carrierwave]: 图片/文件上传。

- [mini_magick]: 图片resize等处理。

- [geocoder]: 地图相关，可以根据地址生成经纬度，也可以根据经纬度来查找地址，判断距离等等。

- [localeapp]: i18n必备的，具体信息查看(http://localeapp.com)。

- [kaminari]: 分页。

- [foreman]: 这个我只能说很cool，很难解释干嘛用的。

- [public_activity]: 记录用户行为。

- [paper_trail]: 记录model改变，这个和上一个gem功能类似，自行取舍。

- [dotenv-rails]: 设置一些变量，比如数据库名称密码地址啊，各个人自己设置互不干扰。

- [responders]: 这也是一个很难说的gem，但是我建议你使用。

- [sidekiq]: 延时任务。

- [whenever]: 定时任务。

- [rails_admin]: 管理员后台快速搭建。

- [jazz_fingers]: 这个gem是irb的增强，集成了pry，awesome_print之类的东西，尤其在rails的console里使用效果不错。（不过发现速度不是很理想，）

- [simple_form]: 继续提高生产力。

- [brakeman]: 聊胜于无的代码安全检查。

- [rubocop]: 代码风格检查（跟rails_best_practices差不多的东西）。

- [rails_best_practices]：跟上面差不多的东西。

- [lograge]: 不错的logger gem。

## 服务器&&Deploy

- [thin]: 一个服务器。

- [unicorn]: 另一个服务器。

- [capistrano]: 一般是部署界的首选。

- [mina]: 另一个部署的选择，会轻量级很多。
