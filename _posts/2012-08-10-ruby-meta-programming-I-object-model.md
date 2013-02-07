---
layout: post
title: Ruby元编程（一）：Ruby的对象模型
---
###元编程简介

Ruby对元编程良好的支持是很大的优势，利用元编程，可以实现很多“魔法”。


  + 连接外部系统时，可以写一个很简单的通用Wrapper，把相应的调用转发给外部系统，而不需要用代码来一一映射。用动态分发和覆盖method_missing的方法都能达到此目的。

  + 可以为特定的用途生成DSL，这种DSL也是Ruby，但看起来就像全新的语言。Rails就是典型代表。

  + 最大化减小重复代码，完美实现DRY原则。依据自定义的规则，从头生成方法、类、模块，甚至任意代码。Java那种烦人的Getter/Setter见鬼去吧！

但“能力越大，责任越大”。强大的工具可以用来建设，也可以搞破坏，甚至能让Ruby运行系统本身崩溃。So, be careful, don't turn to the dark side。

Ruby的元编程属于动态元编程，就是编写在__运行时__操纵语言构件的代码。这有别于C++模板那种只能在编译期生成代码的静态元编程。Ruby的基本系统非常简洁，要用这些砖瓦构建摩天大厦，元编程能力必不可少。

###对象模型

Ruby采用单根的对象系统，所有东西，比如一个数字，或者字符串，都是对象，类和模块本身也是对象。所有对象的根对象都是Object，在1.9之后，在Object之上又引入了BasicObject，这是一个所谓的白板对象，只有非常少的几个方法。在BasicObject之上生成方法，就不用担心和既有方法重名了。

    class MyClass  
    end  
      
    obj = MyClass.new  

上面这段最简单的代码，其对象模型见下图，非常清晰明了吧？

![](http://pic.yupoo.com/merlinran/CCDyNbku/qLJQ6.png)

注意蓝色的方块，那是因为Object类中include了Kernel这个module。Ruby对这个mixin的处理，就是创建一个匿名类来包装这个module，然后插入到继承链中该类的上方。

在irb中，通过ancestors这个方法，也能清晰地看到继承关系。

    1.9.2-p290 :006 >   obj.class.ancestors  
     => [MyClass, Object, Kernel, BasicObject]  
    1.9.2-p290 :007 > obj.class.class.ancestors  
     => [Class, Module, Object, Kernel, BasicObject]  

现在，我们自己定义两个module，再来看看。

    module MyModule  
    end  
      
    module AnotherModule  
    end  
      
    class MyClass  
      include MyModule  
      include AnotherModule  
    end  
      
    obj = MyClass.new  

在mixin时，Ruby会依次为每个module生成一个匿名类，插入继承链上方。因此，这里的AnotherModule就比MyModule后插入链中。

    1.9.2-p290 :014 >   obj.class.ancestors  
     => [MyClass, AnotherModule, MyModule, Object, Kernel, BasicObject]  
    1.9.2-p290 :015 > obj.class.class.ancestors  
     => [Class, Module, Object, Kernel, BasicObject]  

###打开类

在Ruby中，定义一个类，其实只是声明作用域。可以在任何地方声明同一个类，甚至Ruby库里的类。第一次见到类定义时，会新建一个Class对象，再之后，就是对这个对象做修改了。比如在任意地方放入以下代码：

    class String  
      def my_method { blabla... }  
    end  

之后就可以调用"hello, world".my_method了。这种技术，我们称作“打开类”。利用它，可以把第三方的库完美地融入Ruby环境中。

###方法调用

当在对象上调用一个方法时，Ruby首先把self设置为该对象本身，然后找到该对象所属的class，看该类上有没有这个方法，如果没有，就顺着上图中的继承链一直往上找，找到为止，然后以self为对象，调用该方法。这种方式俗称“向右一步，再向上”。有时我们会因此遭遇灵异事件。在上面的例子中，如果MyModule和AnotherModule有同名方法，在MyModule中又调用了该方法，那么调用的其实是AnotherModule中那一个！因为AnotherModule处于继承链的下方，会优先找到它。这种时候，把两条include换个位置，结果就不同了。

如果一直到BasicObject都没找到这个方法，那必定是没有了，于是调用self的method_missing方法。既然是调用方法，那又要循着继承链往上找一遍。覆盖method_missing方法就成了元编程的一个重要手段。

如果是在对象的方法中调用另一个方法呢？那self就是对象本身，这叫做隐式调用。Ruby中的private方法，就是只允许隐式调用的方法。由此可以推断，只有对象自己，才能通过隐式调用，调用自己的private方法，其它情况，即使是同一个类的两个对象，也不行。

如果不在类里面呢？self就对应于一个由Ruby解释器创建的main对象，也叫做top level context。

下一篇，将讲述在Ruby中[如何动态地调用和定义方法](/2012/08/10/ruby-meta-programming-II-dynamic-definition-and-invocation-of-methods.html)。

