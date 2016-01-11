---
layout: post
title: "关于Constant查找的一切(译)"
date: 2016-1-11 14:47:06
categories: ruby
image: /assets/images/chemistry-740453_1280.jpg

---

***注：此文翻译整理自https://cirw.in/blog/constant-lookup，推荐关注，pry主要贡献人***

使用常量很简单，理解如何查找常量的很麻烦。

虽然一句话就可以说清楚常量查找，***依次在Module.nesting, Module.nesting.first.ancestors中查找，如果Module.nesting.first是nil，则在Object.ancestors中查找。***，但是为了更好的理解，我们还是举几个例子吧。


![举几个栗子](/assets/images/for_example.jpg)

### Module.nesting

Ruby首先会根据词法作用域(lexical scope)周围的module或者classes的关联的常量来查找:

```
module A
  module B; end
  module C
    module D
      B == A::B
    end
  end
end
```

上面的例子里在D中的B首先会查找： A::C::D::B (不存在), 然后 A::C::B (不存在), 最后 A::B (找到了)。

跟ruby里的其他东西一样, 查找常量的namespace chains也是在运行时通过Module.nesting可以实现自省的（运行时知道自己的Module.nesting）：

```
module A
  module C
    module D
      Module.nesting == [A::C::D, A::C, A]
    end
  end
end
```

如果你尝试过用简写的声明方式来重打开一个module，你可能已经注意到不使用完整的namespace是无法访问其他常量的。这是因为外面的namespace并没有添加到Module.nesting当中：

```
module A
  module B; end
end

# 错误
module A::C
  B
end

# 正确
module A
  module D
    B
  end
end
# NameError: uninitialized constant A::C::B
```

上面错误的例子中，如果要使用B，那么Module.nesting里必须包含A, 但是上面的nesting里只有[A::C]

### Ancestors

如果在Module.nesting中找不到, Ruby就会在当前打开class或者module的祖先中查找。

```
class A
  module B; end
end

class C < A
  B == A::B
end
```

当前打开是指最里面的class或者module，例如上面查找B的C。一个常见的误解就是常量查找会使用self.class，例如上面也许觉得B应该查找C::B，但实际上并不会这样查找，参考下面的例子。

```
class A
  def get_c; C; end
end

class B < A
  module C; end
end

B.new.get_c #self.class是B，但是并不会查找B::C 当前打开类是A，
# NameError: uninitialized constant A::C
```

### Object::

在顶级作用域的时候，Module.nesting是[]，所以常量直接在当前打开的class和他的祖先里查找，虽然你看不到，但是顶级作用域里的当前打开类是Object。

```
class Object
  module C; end
end
C == Object::C
```

虽然还没有说，但是你也应该注意到，新定义的常量也会在当前打开类里进行定义，所以顶级作用域里定义的常量也会在Object里进行定义：

```
module C; end
Object::C == C
```

这个就解释了为什么在顶级作用域里定义的常量在哪里都可以使用，因为在ruby里大部分类都是Object的子类，所以Object总是在其他类的祖先链里，所以在顶级作用域里定义的常量在其他地方都可以使用。

所以如果你使用过BasicObject(不是Objet的子类)，你会注意到你在BasicObject内部无法访问顶级作用域的常量。

```
class Foo < BasicObject
  Kernel
end
# NameError: uninitialized constant Foo::Kernel
```

如果你想在这种情况下使用，可以使用::Kernel来访问Object::Kernel。

Ruby假设你会把module混入到继承自Object的class之中，所以如果当前打开的是一个module，也会把Object添加到祖先链之中。所以可以在module中使用顶级作用域的常量。

```
module A; end
module B;
  A == Object::A
end
```

### class_eval

正如上面提到的，常量会查找当前打开的类或者module，当前打开取决于周围的class或者module关键字，所以，如果你使用class\_eval或者module\_eval来执行block，这个并不会修改当前打开类。还是会在block定义的作用域内查找。

```
class A
  module B; end
end

class C
  module B; end
  A.class_eval{ B } == C::B
end
```

不过略复杂的事实是，如果你使用字符串来eval，在class\_eval的情况下，Module.nesting中会包含自己的class，在instance\_eval的情况下，Module.nesting中则会包含这个object的singleton class。

```
class A
  module B; end
end

class C
  module B; end
  A.class_eval("B") == A::B
end
```

### 其他细节

最后我想指出，你在一个class的singleton里是无法访问在class内定义的常量。

```
class A
  module B; end
end
class << A
  B
end
# NameError: uninitialized constant Class::B
```

这是因为singleton class的祖先链中并不包含自身的class，直接从Class开始。

```
class A
  module B; end
end
class << A; ancestors; end
[Class, Module, Object, Kernel, BasicObject]
```

相似的情况下，还需要指出已经在Module.nesting里的class的父类并不会出现在Module.nesting之中，如下面的A。

```
class A
  module B; end
end

class C < A
  class D
    B
  end
end
# NameError: uninitialized constant C::D::B
```

### 总结

常量查找并不难，只需要记住查找词法包围的class和module，你可以使用Module.nesting里的第一个值来当作当前打开类或者模块，如果这个值为空，则当前打开类是Object。

下面的代码模拟了ruby内部的常量查找方式，你会注意到我使用字符串作为binding.eval的变量所以Module.nesting是binding的object而不是block。

```
class Binding
  def const(name)
    eval <<-CODE, __FILE__, __LINE__ + 1
      modules = Module.nesting + (Module.nesting.first || Object).ancestors
      modules += Object.ancestors if Module.nesting.first.class == Module
      found = nil

      modules.detect do |mod|
        found = mod.const_get(#{name.inspect}, false) rescue nil
      end
      found or const_missing(#{name.inspect})
    CODE
  end
end
```

(翻译完)

### 其他

实际上常量查找是很好玩的一个过程，可以参考一下Rails实现const\_missing和const\_get的方法。

可以解决诸多疑惑，例如:

```
class Person
end

String.const_get(:Person) # Person
String::Person # Person
String::Person::String # String  
```

