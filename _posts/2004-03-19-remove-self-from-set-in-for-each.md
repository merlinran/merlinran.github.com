---
layout: post
title: 在for_each中调用的方法如何将自己从集合中移除
---
    std::for_each(HookSet_.begin(), HookSet_.end(), CallHook(addr, data, len));
上面这行代码，CallHook是个函数对象，对每个元素调用一个方法。后来发现访问了非法内存，原来在该方法中可能将自己从HookSet_中删除，这样HookSet_的大小就变了。for_each是non-modifying algorithm，在里面干这事是绝对不行的。
怎么办呢？可以保留一个欲删除的Hook集合，UnregisterHook只是将对应的Hook放在这个集合中。for_each完了之后，该集合中就有所有需要删除的Hook，然后用set_difference，就可以将该集合从原集合中排除掉。因此，我尝试这么写：
    HookSetType temp;
    std::set_difference(HookSet_.begin(), HookSet_.end(), HookSetToDelete_.begin(), HookSetToDelete_.end(), back_inserter(temp));
    HookSet_.swap(temp);
把HookSet_和HookSetToDelete_的差集写到temp中，然后将temp和HookSet_交换。

看起来是个好办法，结果编译器告诉我说std::map没有push_back方法。那怎么办呢？预先分配大小!
    HookSetType temp(HookSet_.size());
    std::set_difference(HookSet_.begin(), HookSet_.end(), HookSetToDelete_.begin(), HookSetToDelete_.end(), temp.begin());
    HookSet_.swap(temp);
再次失败。set不是序列容器，不能预设大小。

想来想去，发现用set确实不行。于是转而用vector。幸而Hook的数量不多，不会影响效率。

ACE里好像用的就是这个办法。在某个Event_Handler里可以将自己Unregister掉。Unregister时还会回调一个handle_close()函数，在此函数中可以将自己销毁掉。

其实还有一个办法：

如果这个Hook需要将自己从HookSet_中移除，就在内部设置一个标志。for_each完了之后，再运行：
    std::remove_if(HookSet_.begin(), HookSet_.end(), mem_fun(Hook::OnXXX));
但如果同时还要将Hook彻底删除，标准库里就没有这个东西了，只有用前一种办法。
