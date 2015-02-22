---
layout: post
title:  "简单的长微博示例"
date:   2014-01-22 11:47:06
categories: others
image: /assets/images/post.jpg
---

长微博就是把文字转换成图片，主要用到的gem是[imgkit][imgkit]。

imgkit是一个通过open3来调用wkhtmltoimage生成图片的gem，源码不是很复杂，使用也很简单。

基本步骤如下：

{%highlight ruby%}
rails new long-weibo
#Add imgkit and carrierwave to Gemfile and run bundle
rails g model weibo weibo:string image:string
rake db:migrate
rails g controller home index create
{%endhighlight%}

修改一下routes.rb
{%highlight ruby%}
   post "create" =>'home#create'
   root 'home#index'
{%endhighlight%}
写个简单的界面home/index.html.erb
{%highlight erb%}
<textarea id="content" name="content">
</textarea>
<a id="create" href="#create">Create</a>
<div id="image">
</div>
{%endhighlight%}
写段简单的js
{%highlight javascript%}
$(function(){
  $("#create").click(function(){
     var content = $("#content").val();
     $.post("/create",{content:content},function(returnData){
       $("#image").html("<img src='"+returnData+"'>");
     });
  });
});
{%endhighlight%}
把controller改改
{%highlight ruby%}
  def index
  end
  def create
    @weibo = Weibo.create(weibo:params[:content])
    render :text=>"#{@weibo.image.url}"
  end
{%endhighlight%}
把model改改
{%highlight ruby%}
class Weibo < ActiveRecord::Base
  after_create :generate_image
  mount_uploader :image,ImageUploader
  private
  def generate_image
      file = Tempfile.new(["template_#{self.id.to_s}", 'jpg'], 'tmp', :encoding => 'ascii-8bit')
      #不加html标准标签中文会乱码
      #因为textarea中的换行是\n 需要替换成br标签
      weibo = "<!DOCTYPE html><html lang='zh-cn'><head><meta charset='UTF-8'></head><body>#{self.weibo.gsub("\n","<br />")}</body></html>"
      file.write(IMGKit.new(weibo, quality: 50, width: 600,height:0).to_jpg)
      file.flush
      self.image = file
      self.save
      file.unlink
  end
end
{%endhighlight%}
完成了，超级简陋的长微博生成工具。

源码地址：
[long-weibo][long-weibo]

[long-weibo]:https://github.com/jerry-tao/long-weibo
[imgkit]:https://github.com/csquared/IMGKit
