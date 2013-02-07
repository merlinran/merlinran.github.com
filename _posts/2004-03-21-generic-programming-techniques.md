---
layout: default
title: 翻译：泛型编程技术
---
作者：[David Abrahams](http://daveabrahams.com/)

译者：[Merlin Ran](http://mindhiking.info/)

原文： [http://www.boost.org/more/generic_programming.html](http://www.boost.org/more/generic_programming.html)


摘要：本文大致地介绍一下Boost库中所用到的泛型编程技术

关键字：泛型编程 模板 Traits Tag 类型生成器 对象生成器 策略类
--------------------------------------------------------------------------------
##1、何谓泛型编程

泛型编程（Generic Programming）关注于产生通用的软件组件，让这些组件在不同的应用场合都能很容易地重用。在C++中，类模板和函数模板是进行泛型编程极为有效的机制。有了这两大利器，我们在实现泛型化的同时，并不需要付出效率的代价。

作为泛型编程的一个简单例子，让我们看一下在C库中如何让memcpy()函数泛型化。一种实现方法可能是这样的：

    void\* memcpy(void* region1, const void* region2, size_t n) {
      const char* first = (const char*)region2;
      const char* last = ((const char*)region2) + n;
      char* result = (char*)region1;
      while (first != last)
        *result++ = *first++;
      return result;
    }
这个memcpy()函数已经在一定程度上进行了泛型化，采取的措施是使用void\*，这样该函数就可以拷贝不同类型的数组。但如果我们想拷贝的数据不是放在数组里，而是由链表来存放呢？我们能不能扩展这个概念，让它可以拷贝任意的序列？看看memcpy()的函数体，这个函数对传入的序列有一个_最小需求_：它需要用某种形式的指针来遍历这个序列；访问指针正指向的元素；把元素写到目的地；比较指针以判断何时停止拷贝。C++标准库把这样的需求进行分组，称之为概念（concepts）。在这个例子中就有输入迭代器（对应于region1）和输出迭代器（对应于region2）这两个概念。

如果我们把memcpy()用函数模板重写，使用输入迭代器和输出迭代器作为模板参数来描述对序列的需求，我们就可以写出一个具有较高重用性的copy()函数：

    template OutputIterator copy(InputIterator first, InputIterator last, OutputIterator result) {
      while (first != last)
        *result++ = *first++;
      return result;
    }

使用这个泛型的copy()函数，我们可以拷贝各种各样的序列，只要它满足了我们指定的需求。对外提供了迭代器的链表，比如std::list，也可以通过我们的copy()函数来拷贝：

    int main() {
      const int N = 3;
      std::vector region1(N);
      std::list region2;
      region2.push_back(1);
      region2.push_back(0);
      region2.push_back(3);
      std::copy(region2.begin(), region2.end(), region1.begin());
      for (int i = 0; i < N; ++i)
        std::cout << region1[i] << " ";
      std::cout << std::endl;
    }

##2、何谓概念？

概念就是需求的集合，这些需求可以包含有效表达式、相关类型、不变性，以及复杂度保证。满足概念中所有需求的类型，称之为此概念的一个样例。一个概念可以是对别的概念的扩展，称之为概念的细化。

有效表达式：任意的C++表达式。若某类型是概念的一个样例，那么把该类型的一个对象代入此表达式中，该表达式必能通过编译。

相关类型：与样例相关的一些类型，它们和样例共同出现在一个或多个有效表达式中。如果样例类型是自定义类型，则相关类型可以由类定义中所嵌套的一些typedef来访问。相关类型经常也通过traits类来访问。

不变性：对象的一些运行时特征，必须为真。所有用到这个对象的函数，都必须保持这种特征。不变性通常以前置条件和后置条件的形式出现。

复杂度保证：对概念中的有效表达式执行所需时间或资源所做的限制。

C++标准库中所使用的概念在SGI STL网站上有详细的文档说明。

##3、Traits

traits类为访问编译时实体（类型、整数常量，或者地址）的相关信息提供了一条途径。比如，类模板std::iterator_traits看起来是这个样子：

    template struct iterator_traits { typedef ... iterator_category; typedef ... value_type; typedef ... difference_type; typedef ... pointer; typedef ... reference; };

该traits的value_type可以让泛型代码得知该迭代器所“指向”的是何种类型的数据；而iterator_category对迭代器的能力进行分类，针对不同类型的迭代器我们选择与之适应的最高效的算法。

traits模板有一个很重要的特点：它们是非侵入的（non-intrusive）。我们可以为任何类型提供相关信息，不管它是内建的类型，还是第三方的库中提供的类型。为了给某一特定类型指定traits，一般采用部分特化或者全特化traits模板的方式。

欲更深入的了解std::iterator_traits，可以参阅SGI提供的资料。std::numeric_limits也采用了traits技术，提供表示各内建数值类型取值范围的常量值。

##4、标记分派

另外有一种技术经常与traits合用，那就是标记分派。它依据类型的属性，通过函数重载进行分派。一个很好的例子就是std::advance()函数。这个函数的功能是将一个迭代器递增n次，对于不同类型的迭代器可以有各自优化的实现方法。如果是随机访问迭代器（可以以任意的距离前后跳转），advance()函数可以用i+=n来实现，既简单又高效，只需要常量时间。而其它的迭代器必须一步一步地递增，需要线性的时间复杂度。如果是双向迭代器，n就可能为负值，因此必须判断对迭代器到底是增还是减。

标记分派和traits类的联系很紧密。分派所依据的属性（在这个例子中是iterator_category）一般都是通过traits类来取得。主advance()函数从iterator_traits中获得对应于该iterator的iterator_category，然后调用重载过的advance_dispach()函数。编译器依据作为参数传给advance_dispach()的iterator_category，选择合适的重载版本。标记只是一个极其简单的类，它的唯一任务就是为标记分派或者其它类似技术传递必要的信息。

    namespace std {

      struct input_iterator_tag { };

      struct bidirectional_iterator_tag { };

      struct random_access_iterator_tag { };

      namespace detail {

        template void advance_dispatch(InputIterator& i, Distance n, input_iterator_tag) {
          while (n--) ++i;
        }

        template void advance_dispatch(BidirectionalIterator& i, Distance n, bidirectional_iterator_tag) {
          if (n >= 0)
            while (n--) ++i;
          else
            while (n++) --i;
        }

        template void advance_dispatch(RandomAccessIterator& i, Distance n, random_access_iterator_tag) {
          i += n;
        }

      }

      template void advance(InputIterator& i, Distance n) {
        typename iterator_traits::iterator_category category;
        detail::advance_dispatch(i, n, category);
      }
    }

##5、适配器

适配器是一种类模板，建立在其它类型之上，提供新的接口或者行为。标准库中就使用了适配器，比如std::reverse_iterator通过反转迭代器的递增/递减行为，适配了迭代器，还有std::stack，通过适配标准容器，提供一个简单的堆栈接口。

在这里可以找到标准库中所用适配器的深入阐述。

##6、类型生成器

类型生成器的工作是依据它的模板参数合成新的类型[注]。类型生成器产生的新类型一般作为生成器中嵌套的typedef出现。用类型生成器的主要目的是为了让复杂的类型表达式显得更简单些。比如boost::filter_iterator_generator：

    template struct filter_iterator_generator {
      typedef iterator_adaptor< Iterator,filter_iterator_policies, Value,Reference,Pointer,Category,Distance> type;
    };

看起来可真够复杂的。但现在生成一个合适的filter iterator可就容易多了，只需要写： boost::filter_iterator_generator::type 就行了。

##7、对象生成器

对象生成器是一种函数模板，依据其参数产生新的对象。可以把它想象成泛型化的构造函数。有些情况下，欲生成的对象的精确类型很难甚至根本无法表示出来，这时对象生成器可就管用了。对象生成器的优点还在于它的返回值可以直接作为函数参数，而不像构造函数那样只有在定义变量时才会调用。Boost和标准库中用到的对象生成器大多都加了个前缀make_，比如std::make_pair(const T&, const U&)。

看看下面的例子：
    struct widget {
      void tweak(int);
    };
    std::vector widget_ptrs;

通过把两个标准的对象生成器bind2nd和mem_fun合用，我们可以很轻松地tweak所有的widget：
    void tweak_all_widgets1(int arg) {
      for_each(widget_ptrs.begin(), widget_ptrs.end(), bind2nd(std::mem_fun(&widget::tweak), arg));
    }

如果不用对象生成器，上面的函数可能就得这样来实现：
    void tweak_all_widgets2(int arg) {
      for_each(struct_ptrs.begin(), struct_ptrs.end(), std::binder2nd<std::mem_fun1_t >( std::mem_fun1_t(&widget::tweak), arg));
    }
表达式越复杂，就越需要缩短这些冗长的类型说明，对象生成器的好处也就越能显现。

##8、策略类

策略类就是用来传递行为的模板参数。标准库中的std::allocator就是策略类，把内存管理的行为应用到标准容器中。

Andrei Alexandrescu在他的文章中对策略类进入了全面而深入的分析，他写道：
   策略类精确地反映了设计时的抉择。他们要么从其它类中继承，要么包含在其它类中。策略类在同样的句法结构上提供了不同的策略。使用策略的类把它用到的每一个策略都作为模板参数，这样，用户就可以自由地选择需要使用的策略。
 
    策略类的强大在于它们能够自由地组合在一起。通过把策略类作为模板参数的办法来组合多种策略，代码量与使用的策略数只成线性关系。

Andrei认为策略类的强大源于其小粒度和正交性。Boost在迭代适配器库中的策略类运用可能淡化了这一卖点。在这个库中，所有已适配的迭代器的行为都放在一个策略类里面。其实，Boost并不是开先河者。std::char\_traits就是一个策略类，它决定了std::basic\_string的行为，尽管它的名字叫traits而不叫policy。

注：因为C++缺少模板化的typedef，类型生成器可以作为其替代方案。

Revised 14 Mar 2001

Copyright David Abrahams 2001.

Permission to copy, use, modify, sell and distribute this document is granted provided this copyright notice appears in all copies. This document is provided "as is" without express or implied warranty, and with no claim as to its suitability for any purpose.

翻译于2003年5月9日

版权声明：和David Abrahams的一样:-)
