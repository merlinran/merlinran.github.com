---
layout: post
title: Digging into Boost：在 VC7 中如何实作添加/去除reference的template？
---
之所以对这个感兴趣，主要是为了回答一篇[同名帖子](http://expert.csdn.net/Expert/topic/1704/1704020.xml?temp=.1142847)。

障碍主要是VC7不支持模板部分特化。我手头的VC6同时也不支持整型static const成员的直接初始化，不知VC7支持与否。

在[这个文件](http://www.boost.org/boost/type_traits/is_reference.hpp)里看到了它采用的方法。主要部分如下：

    namespace detail {
    using ::boost::type_traits::yes_type;    //typedef char yes_type
    using ::boost::type_traits::no_type;     //struct no_type{ char padding[8]; }
    using ::boost::type_traits::wrap;          //template <class T>class wrap{}

    template <class T> T&(* is_reference_helper1(wrap<T>) )(wrap<T>);//[1]
    char is_reference_helper1(...);                                                                  //[2]

    template <class T> no_type is_reference_helper2(T&(*)(wrap<T>));  //[3]
    yes_type is_reference_helper2(...);                                                          //[4]

    template <typename T>
    struct is_reference_impl
    {
        BOOST_STATIC_CONSTANT(
            bool, value = sizeof(
                ::boost::detail::is_reference_helper2(
                    ::boost::detail::is_reference_helper1(::boost::type_traits::wrap<T>()))) == 1    //*****
            );
    };//namespace detail

    BOOST_TT_AUX_BOOL_TRAIT_DEF1(is_reference,T,::boost::detail::is_reference_impl<T>::value)//=====

[1]定义了一个函数is_reference_helper1，它以一个wrap<T>为参数，返回一个函数指针，这个指针指向的函数以wrap<T>为参数，返回一个T&。如果未来的编译器支持template typedef的话（一个刚刚才提交给standard committe的proposal），就等价于：

    template<class T> typedef T& FUNC(wrap<T>);
    template<class U> FUNC<U>* is_reference_helper1(wrap<U>);

参见标准文档8.3.5[9]的最后一个例子（在看时请记住：如果不指定返回值，默认的返回值为int）。

[3]接受一个同[1]返回类型一致的函数指针作为参数，返回no_type。

[2]和[4]则是在[1]和[2]不匹配时被匹配，因为它们的参数是...，可以匹配任意参数。

关键在于\*\*\*\*\*那一行：如果T是引用类型，因为不允许引用的引用，所以在[1]和[2]中，[1]不匹配，会选择[2]，返回char，然后选择[4]，返回yes_type。因为yes_type实际上就是char，所以sizeof(yes_type)就是1。
 
注意，这里的几个函数都只有声明，并没定义（实际上也没办法定义）。如果要依靠函数定义，那就涉及到运行期了。有了函数的声明，就知道了它的返回类型，根据这个类型，便可以选择下一步匹配的函数，也可以对它取sizeof()。这个技巧常常用到。

BOOST_NO_INCLASS_MEMBER_INITIALIZATION的定义在[这里](http://www.boost.org/boost/config/suffix.hpp)，只是一个简单的开关。

如果T是引用类型，BOOST_STATIC_CONSTANT展开后应该是static const bool value=sizeof(yes_type)==1，于是is_reference_impl<T>::value==true。

但VC6同样又不支持integral static const member的in class initialization，所以BOOST用enum来代替，展开后就是：

    template <typename T>
        struct is_reference_impl
    {
            enum {value = sizeof(yes_type)==1};
    };

在实现中，二者一般可以互换。

BOOST_TT_AUX_BOOL_TRAIT_DEF1（带=====那一行）所做的事比较多，主要是在值与类型之间变换。涉及到MPL库，我水平不够，对它还没有研究。其实如果只考虑这里的实现，只需要写个模板：

    template <typename T>is_reference{
              static const bool value = is_reference_impl<T>::value;
    };

把对is_reference引用直接转移给 is_reference_impl。enum的处理也类似。

现在已经判断出T是不是引用类型，下面的工作就轻松了。

以下从[http://www.boost.org/boost/type_traits/add_reference.hpp](http://www.boost.org/boost/type_traits/add_reference.hpp)中摘录。注意，是add_reference：

    template <bool x>
    struct reference_adder
    {
        template <typename T> struct result_
        {
            typedef T& type;
        };
    };

    template <>
    struct reference_adder<true>
    {
        template <typename T> struct result_
        {
            typedef T type;
        };
    };

    template <typename T>
    struct add_reference_impl
    {
        typedef typename reference_adder<
              ::boost::is_reference<T>::value
            >::template result_<T> result;

        typedef typename result::type type;
    };

同样，可以定义：

    template<typename T> struct add_reference{
            typedef typename ::boost::detail::add_reference_impl<T>::type type;
    };

根据is_reference返回的结果，选择泛化版本或者（全）特化版本的reference_adder。而reference_adder里面又有个内嵌类区分T和T&，这样就绕开了部分特化的障碍。

虽然reference_adder的模板参数指定是bool，但任何整型值（包括enum）也可以用来实例化这个模板，非0值转化为true，0值转化为false。这和以类型作为模板参数的情况是不同的，那种情况需要精确匹配（参见前面的[1]和[3]）。

那么去除引用怎么实现呢？我们来看看[这个文件](http://www.boost.org/boost/type_traits/remove_reference.hpp)吧：

    BOOST_TT_AUX_TYPE_TRAIT_DEF1(remove_reference,T,typename detail::remove_reference_impl<T>::type)

把宏展开来：

    template <typename T> struct remove_reference{
            typedef typename detail::remove_reference_impl<T>::type type;
    };

    namespace detail{
        template <typename T> remove_reference_impl{ typedef T type; };
    };

实际上也就是：

    template <typename T> struct remove_reference{
            typedef T type;
    };

这是什么意思？下面是[http://www.boost.org/libs/type_traits/index.htm#transformations](http://www.boost.org/libs/type_traits/index.htm#transformations)中关于::boost::remove_reference<T>::type的Compiler requirements：

  If the compiler does not support partial-specialization of class templates, then this template will compile, but will have no effect, except where noted below.

原来根本不起作用，于是在VC6中实现不了通用的remove_reference。
 
由此可见，部分特化在type_traits库中扮演了多么重要的角色！如果编译器实现了部分特化，只需要写简单而优雅的代码：

    template <typename T> struct remove_reference     { typedef T type; };
    template <typename T> struct remove_reference<T&> { typedef T type; };

而实现这样的功能，在没有部分特化的情况下，却做不到！
 
泛型编程在很多情况下都要对类型进行运算，在缺少部分特化的环境里，就处处受制。通过sizeof()把类型运算转化为值运算，依据值就可以进行模板的全特化，从而绕过了这个障碍。

PS：这是我发的第一篇技术文章，原本打算直接贴在讨论组，但实在太长了，就想到了往这里放。请大家指正。
