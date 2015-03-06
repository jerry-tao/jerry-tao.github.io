---
layout: post
title: "Angular Best Practices"
date: 2015-3-6 11:47:06
categories: Angular
image: /assets/images/glasses.jpg

---

本文所有内容来源于http://www.meetup.com/AngularJS-MTV/events/93943412/ （我个人仅仅是整理学习一下:D）。

(本文在我的github上存储，欢迎P/R)

视频地址：https://www.youtube.com/watch?v=ZhfUv0spHCY

slide地址：http://goo.gl/CD0Is

Version: 1.0.0@15-3-6

## 1. 建立项目


- yeoman
- 参考[angular-seed](https://github.com/angular/angular-seed)

## 2. 加载javascript



- 在HTML下面使用script标签。
- AMD(requirejs) 如果使用这个你必须在加载完js之后手动设置一下ngApp位置：

[angluar-seed/index-async.html](https://github.com/angular/angular-seed/blob/master/app/index-async.html#L41)

总结：其实大多数angular app都是SPA， 所以一次加载js的方式即可。

## 3. Zen Of Angular



**内容显示**

使用\{\{\}\}这种方式来显示值会有问题，有些情况下index.html会闪现并且用户会看到\{\{\}\}部分。

解决办法：

- 使用ng-cloak和directive来隐藏\{\{\}\}。
- 使用ng-bind来代替\{\{\}\} 只需要在index.html中使用，动态加载的内容会先预编译再显示，所以没有这个问题。

**整理js**

缩小和混淆js的时候注意一下页面的变量还有inject是依赖于名称的，最好不要随便修改变量名称。

###  Don't fight the html, extend it.



**Templates**

不支持IE的清空下 `<my-directive>` 这种带namespace的命名方式是推荐的做法。

考虑支持IE的情况下 使用`<div my-directive>`, 这种情况IE可以正常的处理，仅仅使用`<my-directive>`不可以。

考虑支持验证HTML，还可以再加个前缀`<div x-my-directive>`或者`<div data-my-directive>`

###  Separate presentation and business logic.



**Controllers**

- 不应该包含DOM的引用。
- 应该响应View的行为：点击按钮，或者提交表单。

**Services**

- 不应该包含DOM的引用。
- 行为不应该跟view有关。

所有对DOM的操作应该放进directive，

在这里的核心思想是，controller应该尽可能的简单，仅仅是把其他的东西bind到一起，例如如何在一个controller里调用另一个controller的方法，这种操作应该是service的，而不应该是controller的。

**Scopes**

- $scope 在view里只读
- $scope 在controller里只写

这个很好理解，例如定义一个addTodo 函数在controller里

```javascript
$scope.addTodo = function(){ # Do something with $scope.todo}

$scope.addTodo = function(todo){ # Do something with todo}
```

第一个你是直接在view里面对$scope赋值，然后在controller里处理，违反了view里只读和controller里只写的原则。

所以第二种是合适的做法。第二种的优点包括更容易理解，更容易测试，更容易复用等。

还有ng-model在继承scope里的表现很值得注意，因为基本上所有的directive都会创建一个自己的$scope，所以使用ng-model在子model中很可能会创建一个自己的$scope变量，而不是使用父$scope的。

所以，在使用ng-model的时候，永远使用一个'.'。

eg: `<input ng-model='title'>` 不正确，应该使用 `<input ng-model='post.title'>`， 这样可以避免子scope创建自己的title。

**modules**

- 根据功能来组织module，而不是type，例如不是把所有的controllers来组织到一个module里。
- 根据view来组织module是一个很好的习惯。

**PS: Develop with non-minified libraries**
