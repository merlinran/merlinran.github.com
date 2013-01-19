---
layout: default
title: 关于结构化程序设计、OO与设计模式的讨论
---
2006-10-27  22:15:30  Joe Li  Merlin Ran, just do it  我觉得用C++或其他对象特性语言，没有很好的设计模式，简直是误用．

2006-10-27  22:16:10  Merlin Ran, just do it  Joe Li  用什么语言都有模式，这是我后来发现的

2006-10-27  22:18:49  Merlin Ran, just do it  Joe Li  比如有一个原则：程序逻辑最好由数据驱动，而不要用代码驱动，这在C编程里就讲得比较多。

2006-10-27  22:20:18  Joe Li  Merlin Ran, just do it  Ｃ这样的过程语言里面，我觉得就简洁多了，结构化程序设计，自顶向下，足不求精，模块化．这很明显．我觉得不需要其他什么模式，过分强调模式就过分了．

2006-10-27  22:21:02  Merlin Ran, just do it  Joe Li  如果结构化程序设计能够满足要求，OO就不会这么大行其道了

2006-10-27  22:21:33  Joe Li  Merlin Ran, just do it  ｏｏ就是折腾

2006-10-27  22:21:49  Joe Li  Merlin Ran, just do it  Data dominates. If you've chosen the right data structures and organized things well, the algorithms will almost always be self-evident. Data structures, not algorithms, are central to programming. - Fred Brooks 

2006-10-27  22:22:07  Merlin Ran, just do it  Joe Li  现在很多操作系统内核的设计不都是面向对象的吗？包括Windows，包括BSD。虽然不用OO语言，但无处不透露着OO的思想在里面。

2006-10-27  22:22:11  Joe Li  Merlin Ran, just do it  我觉得这里的意思，不是说模式。

2006-10-27  22:22:15  Joe Li  Merlin Ran, just do it  http://www.google.com/notebook/public/04843391034506425457/BDRihIgoQ-fe8sdgh

2006-10-27  22:23:17  Joe Li  Merlin Ran, just do it  ＞虽然不用OO语言，但无处不透露着OO的思想在里面。 那你说它是怎么透露的，由那些表现和证据呢。

2006-10-27  22:23:57  Joe Li  Merlin Ran, just do it  那个链接里有Fred Brooks 那句话，所以发了一下给你　:)

2006-10-27  22:26:01  Joe Li  Merlin Ran, just do it  ｄｍｒ现在在主持开发ａｔｔ的ｐｌａｎ９类似ｕｎｉｘ的操作系统，用的是Ｃ。

2006-10-27  22:26:03  Merlin Ran, just do it  Joe Li  比如Windows的HANDLE或者UNIX的file descriptor，你可以open、read、write、close，甚至可以不用知道它是一个物理文件、PIPE，还是socket

2006-10-27  22:28:05  Merlin Ran, just do it  Joe Li  你也关心过Plan 9啊？很多想法是非常好的，不过由于与既有系统差别太大了，成功推广的可能性不太大吧。

2006-10-27  22:31:14  Joe Li  Merlin Ran, just do it  偶尔看了一下．应该是说创新吧，要不有ｕｎｉｘ挡在前面，好意思做个玩具嘛。ｕｎｉｘ也不是用于推广的:)

2006-10-27  22:32:33  Joe Li  Merlin Ran, just do it  我好像看过有本书说，ｕｎｉｘ把很多东西都当文件，所以可以用ｏｐｅｎ，ｃｌｏｓｅ这些操作。我觉得这些与ｏｏ没什么联系。

2006-10-27  22:32:38  Merlin Ran, just do it  Joe Li  还有一个原因应该是前期没有开放，导致研究它的人太少

2006-10-27  22:32:51  Merlin Ran, just do it  Joe Li  这不是多态吗？

2006-10-27  22:33:25  Merlin Ran, just do it  Joe Li  哦，不能叫多态，顶多类似于继承

2006-10-27  22:35:02  Merlin Ran, just do it  Joe Li  还有比如调页算法的实现。对不同类型的页面，指定不同的算法，其实就是附加一个不同的函数指针。需要调页时，只需要统一地调用这个函数指针就行了。

2006-10-27  22:35:16  Merlin Ran, just do it  Joe Li  这个类似于多态。

2006-10-27  22:37:50  Joe Li  Merlin Ran, just do it  我觉得不很似。我找不到那本书了。

2006-10-27  22:39:15  Merlin Ran, just do it  Joe Li  多态不就是同一套操作，当应用于不同对象时，表现出不同的行为吗？

