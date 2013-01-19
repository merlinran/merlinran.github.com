---
layout: default
title: 从一小段代码看宏的使用技巧
---
虽然C++中应该极力避免宏，但有时候，宏是不可避免的。还是有一些技巧，可以尽量减小使用宏所带来的问题。

下面这段代码来自ACE：

    #define ACE_ERROR(X) do {
      int __ace_error = ACE_Log_Msg::last_error_adapter ();
      ACE_Log_Msg *ace___ = ACE_Log_Msg::instance ();
      ace___->conditional_set (__FILE__, __LINE__, -1, __ace_error); ace___->log X;
    } while (0)

在这段短短的代码中，就有两个很有用的技巧：

1. 用do { } while (0)来包含多行的宏。

    这样有多个好处：
        1. 在do { } while (0)块中定义的局部变量的生命期不会延续到大括号作用域以外。
        2. 在使用宏的时候，尾部加不加括号都能够编译通过。
        3. 逻辑上将宏分隔开来，增加可读性。

2. 宏的“重载”

我们都知道，宏是不能重载的，也就是说，不能这样： #define ACE_ERROR(X) ... #define ACE_ERROR(X, Y) ... 但有时就需要这种重载，特别是需要类似于printf()那样的可变参数时，用处很大。这段代码就解决了这个问题。请看倒数第二行：
    ace___->log X;
很有意思，log和X之间留有一个空格，这可不是C++的语法呀。但宏是将参数原样替换的，所以，如果以ACE_ERROR((LM_ERROR, "Oops!")) 来调用宏（注意这里的双重括号，没有双重括号，是编译不通的），宏参数中的X，将被替换为(LM_ERROR, "Oops!")，进而，上面那行将会替换为：
    ace___->log (LM_ERROR, "Oops!");
这样，就奇妙地变成了函数调用。

在VCL的C++版本中，有个宏叫ARRAYOFCONST，也需要加两重括号，当前百思不得其解，现在终于明白其用意了。
