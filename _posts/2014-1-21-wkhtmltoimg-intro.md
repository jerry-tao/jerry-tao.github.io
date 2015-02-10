---
layout: post
title:  "wkhtmltoimage简介"
date:   2014-01-21 11:47:06
categories: others
image: /assets/images/post.jpg
---

最近有个需求需要从网页生成图片，找到了这么个东西，wkhtmltoimage，这是一个基于webkit来把网页生成图片的开源项目，可惜的是文档少的可怜，所以简单总结一下。

通过URL生成图片：
`wkhtmltoimage http://www.baidu.com/ ~/baidu.jpg`

通过本地html生成图片：
`wkhtmltoimage ~/index.html ~/index.png`

关于HTML中引用的资源介绍：

只要你通过浏览器打开你要生成图片的HTML，如果所有资源都正常被引用了，那么在生成的时候就不会丢失。



关于HTML中CSS样式介绍：

因为是基于webkit的，所以跟chrome打开的效果最接近。



关于HTML中js load的介绍：
有很多网页有很多图片等资源是后加载进来的，为了测试这种情况我们试一下jd的首页：
`wkhtmltoimage http://www.jd.com ~/jd.png`

打开之后可以发现所有图片资源都没有正常显示，关于这点大家要注意一下。


关于生成图片的尺寸：
`wkhtmltoimage --height 200 --width 400  http://www.baidu.com/ ~/baidu-small.png`

这个height跟width类似你用浏览器访问，然后resize浏览器到这个大小的时候的显示情况。


关于生产图片的质量：
`wkhtmltoimage --quality 30 http://www.baidu.com ~/baidu-quality.png`

这个可以设置生成图片的质量，默认是94,取值范围是0～100.

关于生成图片的大小：
有些时候我们不想对整个网页截图，所以需要设置一些剪切效果。
`wkhtmltoimage --crop-h 20 --crop-w 20 --crop-x 0 --crop-y 0 http://www.baidu.com/ ~/bd-small.png`

这个会生成一个高20宽20的以（0,0）为起点坐标的正方形图片。

关于生成图片的格式：

默认支持的格式包括PNG，JPG，TIFF，基本涵盖了常见的图片格式，通过后缀就可以实现。

同时，还有一个wkhtmltopdf的项目，跟这个使用方式类似。



wkhtmltoimage的下载地址：
[wkhtmltoimage][wkhtmltoimg]
[wkhtmltoimg]:http://code.google.com/p/wkhtmltopdf/downloads/detail?name=wkhtmltoimage-0.11.0_rc1-static-amd64.tar.bz2
