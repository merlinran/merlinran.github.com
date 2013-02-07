---
layout: post
title: Ruby元编程（二）：方法的动态调用和定义
---
承上篇，[Ruby的对象模型](/2012/08/10/ruby-meta-programming-I-object-model.html)。

###动态调用

用Object#send方法，可以在运行时调用对象的任意方法。有这项利器，一条语句就可以达到静态语言要用一堆switch来实现的功能。

比如某个parser的输入像这样：

    string "hello, world"  
    integer 10  
    float 3.14  
    ...  

而你的Parser对应就有parseString，parseInteger，parseFloat等方法。静态语言的实现可能是这样：

    switch (format) {  
      case "string": parser.parseString(content);  
      case "integer": parser.parseInteger(content);  
      case "float": parser.parseFloat(content);  
      ...  
    }  

而用Ruby就简单了，一条语句

    parser.send("parse#{format.capitalize}", content)  

###动态定义

在上面的例子中，尽管做到了动态调用，但仍要定义Parser类仍需要定义parseXXX等一大堆方法，而这些方法做的事情可能都差不多，比如往一个XML文件里加入一个以format命名，以content为值节点。其实，我们可以利用Object#define_method，动态地定义方法来消除这种重复。

    require 'rexml/document'  
    include REXML  
      
    class Parser  
      def initialize  
        @root = Document.new.add_element "root"  
      end  
      
      def self.define_parser(format)  
        define_method("parse#{format.capitalize}") { |content|  
          el = Element.new format  
          el.add_text content.to_s  
          @root.add_element el  
        }  
      end  
      
      %w(string integer float).each { |name| define_parser name }  
      
      def dump  
        @root.write $stdout  
      end  
    end  
      
    parser = Parser.new  
    parser.parseString("hello, world")  
    parser.parseInteger(123)  
    parser.parseFloat(3.14)  
    parser.dump  

在上面的代码中，Parser.define_parser可以根据传给它的任意名字，生成名叫parseXXX的实例方法。我们在第17行，利用字符串数组，给它动态定义了三个parse方法。这段代码的输出如下：

    <root><string>hello, world</string><integer>123</integer><float>3.14</float></root>  

###幽灵方法

上面讲的方式还是不够灵活。假如某天输入里增加了一种类型，比如date，我们却没有预先定义对应的方法，那肯定会出错了。

    1.9.2-p290 :029 > parser.parseDate(Date.new)  
    NoMethodError: undefined method `parseDate' for #<Parser:0x8789940 @root=<root> ... </>>  

应对这种情况，我们就要采用响应式的方法，如果某个方法不存在，就无中生有，变一个出来。这就要覆盖前一篇中讲到的method_missing了。新的代码如下：

    require 'rexml/document'  
    require 'date'  
    include REXML  
      
    class Parser  
      def initialize  
        @root = Document.new.add_element "root"  
      end  
      
      def method_missing(id, *args, &block)  
        super if id !~ /^parse(.+)/   # only handle parseXXX  
        el = Element.new $1           # $1 is the last match result of (.+)  
        el.add_text args[0].to_s  
        @root.add_element el  
      end  
      
      def respond_to?(method)  
        method.to_s =~ /^parse.+/ || super  
      end  
      
      def dump  
        @root.write $stdout  
      end  
    end  
      
    parser = Parser.new  
    parser.parseString("hello, world")  
    parser.parseInteger(123)  
    parser.parseFloat(3.14)  
    parser.parseDate(Date.new)  
    parser.dump  

这种技术，我们叫它“幽灵方法”。在代码中我们不仅定义了method_missing，还对应定义了respond_to?，不然就太诡异了：一个对象不支持的方法，竟然还能调用？

代码输出如下：

    <root><String>hello, world</String><Integer>123</Integer><Float>3.14</Float><Date>-4712-01-01</Date></root>  

