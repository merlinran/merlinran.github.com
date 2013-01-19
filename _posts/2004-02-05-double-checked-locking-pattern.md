---
layout: default
title: Double Checked Locking模式
---
该模式由ACE的创建者Doug C. Schimdt提出，在ACE中得到了很广泛的应用。[有专文论述](http://www.cs.wustl.edu/~schmidt/PDF/DC-Locking.pdf)

用代码表示如下：
    static MyObject::Instance() {
      if (thePointer == 0) {
        Guard g(&theMutex);
        if (thePointer == 0)
          thePointer = new MyObject;
      }
      return thePointer;
    }
一看就知道，这是一个Singleton模式。为了避免多线程访问时出现Race Condition，所以引入了同步机制。但如果每次访问都要同步，那会有很大的开销，所以这里进行了两次判断。如果thePointer != 0，也即对象已经建立，就直接返回了。只有当该单件还未创建时，才需要进入临界区内。

但有很多人对此有争议。下面是在新帆新闻组上一位大侠的阐述：

    "wang tianxing" 写入邮件 news:0b23ovg363i5eob7pgodlpj9mcq787i0u7@4ax.com... 
    > On 30 Sep 2003 09:13:15 +0800, "Merlin Ran" wrote: 
    > 
    > 多种原因。比如，创建对象的线程给 thePointer 赋值到一半的 
    > 时候，另一个线程执行到那个 if，发现 thePointer 不是 0， 
    > 因此立即返回 thePointer 的值，但此时 thePointer 里的值只 
    > 有半个。 
    > 
    > 上面说的问题还不算致命的，可能可以用 sig_atomic_t 之类的 
    > 标志来避免非原子的赋值操作。从这个问题里，至少要了解多线 
    > 程时的各种细微的安全问题。 
    > 
    > 真正致命的问题是，一个 CPU 修改的内存，另一个 CPU 如果不 
    > 使用专门的 cache 同步指令，就不保证能够看见，也不保证看 
    > 不见。因此可能发生这样的情况，thePointer 是正确的，指向 
    > 一个 MyObject 对象，但在执行 if() 语句的那个 CPU 上，可 
    > 能看到了 thePointer，但却没有看到它所指的那片存储区上已 
    > 经被初始化好了的 MyObject 对象。 
    > 
    > 这种多个 CPU 的 cache 间的不一致性只发生在多 CPU 机器上， 
    > 不过现在多 CPU 机器已经用的相当普遍，应该考虑程序的普遍 
    > 正确性和一点性能优化之间的取舍。 
    > 
    > 在没有那个 if 的情况下，每次调用操作系统的互斥原语时，调 
    > 用 CPU 的 cache 会被刷新，这是操作系统应该做的事情之一。 
    > 因此不会出现问题。 
他说的第一个问题，可以解决。如果是int，指针等基本原语类型，而且对这些类型的赋值是原子操作，还不需要用sig_atomic_t。当然，这就行针对不同平台进行考量。

而第二个问题，可以将thePointer声明为volatile，这样，编译器每次读写，都会直接读写内存，而不会用寄存器或者Cache来缓存，避免了此问题。

下面这篇文章对第二个问题有比较详细的解释：<http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html>

但是，在[下面的新闻组帖子](http://groups.google.com/groups?hl=zh-CN&lr=lang_zh-CN|lang_zh-TW|lang_en&ie=UTF-8&oe=UTF-8&threadm=MPG.191efc0a3bf86e2e9896e0%40news.hevanet.com&rnum=2&prev=/groups%3Fq%3Ddouble%2Bchecked%2Blocking%26hl%3Dzh-CN%26lr%3Dlang_zh-CN%257Clang_zh-TW%257Clang_en%26ie%3DUTF-8%26oe%3DUTF-8%26selm%3DMPG.191efc0a3bf86e2e9896e0%2540news.hevanet.com%26rnum%3D2)中，Scott Meyers又提出了新的问题：编译器可以对代码进行优化，而这种优化，是程序员无法控制的。

    On Sat, 03 May 2003 20:52:04 +0000, Scott Meyers wrote: 
    > As an example of one of the "more obvious" reasons why it doesn't work, consider 
    > this line from the above code: 
    > 
    > pInstance = new MySingleton; 
    > 
    > Three things must happen here: 
    > 1. Allocate enough memory to hold a MySingleton object. 
    > 2. Construct a MySingleton in the memory. 
    > 3. Make pInstance point to the object. 
    > 
    > In general, they don't have to happen in this order. Consider the following 
    > translation. This isn't code a human is likely to write, but it is a valid 
    > translation on the part of the compiler under certain circumstances (e.g., when 
    > static analysis reveals that the MySingleton constructor cannot throw): 
    > pInstance = // 3 
    > operator new(sizeof(MySingleton)); // 1 
    > new (pInstance) MySingleton; // 2 
    > 
    > If we plop this into the original function, we get this: 
    > >
    > MySingleton *MySingleton::Instance(void) 
    >
    > { 
    > > if(!pInstance) // Line 1
    > > {
    > > LOCK(); // Do some MT-locking here
    > > if (!pInstance) 
    > pInstance = 
    > operator new(sizeof(MySingleton)); // Line 2 
    > new (pInstance) MySingleton;
    > > UNLOCK();
    > > return sp;
    > > } 
    > 
    > So consider this sequence of events: 
    > - Thread A enters MySingleton::Instance, executes through Line 2, and is 
    > suspended. 
    > - Thread B enters MySingleton::Instance, executes Line 1, sees that pInstance 
    > is non-null, and returns. It then merrily dereferences the pointer, thus 
    > referring to memory that does not yet hold an object. 
    > 
    > If there's a portable way to avoid this problem in the presence of aggressive 
    > optimzing compilers, I'd love to know about it.

不仅编译器可以优化，CPU也可以优化，可以调整指令顺序(乱序发射)，以达到最大化的并行处理。 我感觉，Double Checked Locking模式不能称为模式，它只是一种小技巧，有很多条件限制。

但在ACE的代码，包括JAWS中，都用到了这个模式，而且在JAWS中根本没有采取任何防护措施。赋值的对象不是原语类型，也没有用volatile修饰。在ACE的邮件列表中，有过很大的争议，但最后好像不了了之。
