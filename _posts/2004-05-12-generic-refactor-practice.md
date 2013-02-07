---
layout: post
title: 泛型重构实践
---
重构的概念和编程一样古老，但长久以来，人们并未把它作为一项专门的技术提炼出来进行研究。随着Martin Fowler著作的面世，以及XP方法的广为传播，重构思想开始深入人心，相关的书籍和杂志文章也层出不穷，但焦点大多集中在OO代码上。而针对C++，也包括Java和C#中应用日益广泛的模板，却鲜有经验之谈。其实，泛型代码又何尝不需要重构？而且是更需要重构，因为它广泛存在于库中，需要不断提高整体质量。

笔者在此借用一个具体开发中的简单实例，根据需求的变化，对它不断进行重构，希望能够让大家有一些感性认识。同时，也期望抛砖引玉，引起大家对这个领域的重视，逐渐形成一些对泛型代码进行重构的固定手法。

###问题描述：

本项目是一个企业级的网络访问控制软件，安装在局域网的出口处，可以控制局域网中的每台计算机能否访问某个特定的网页，能否聊天、打游戏，或者收发邮件等等，并把协议流量都记录下来以供分析。国际上的通用称谓是雇员Internet管理（EIM）软件，主要目的是减少员工在上班时间从事与工作无关的活动，提高工作效率。

其中，网页过滤是一个比较重要的模块，它需要把经网卡捕获到的每个URL请求，在一个存放着数百万条URL的数据库中查找，再决定是不是要切断连接。数据库如此庞大，大部分内容都只能存放在辅存中。如果每次都要访问硬盘查找，速度很难达到要求。但大家平时上网都能感觉到，一个网页中的链接大多数也都是指向同一个网站，如果能够将此前访问过的URL记住，以备后查，是个不错的主意。我们把这个玩意儿叫做Cache，是取其高速缓存之意。实际的URL数据库和Cache都采用字符串部分匹配，但为了简单起见，在这里只取主机名进行精确匹配。

对这个Cache，定几个的基本要求：

  + 固定大小；

  + 查找时间有上限；

  + 使用最近最少使用（LRU）作为替换算法。之所以选择LRU算法，在教科书里都讲过，因为它是堆栈型算法，命中率会随容量单调增加。

###第一步，根据具体需求，快速准确地完成目标

让我们根据前面定出的要求编写测试。为了简单起见，我并没有使用CppUnit或者Boost.Test等测试框架，而只使用了标准C++具备的断言。对这么个简单的东西，没有必要拿大炮打蚊子，而且，项目组也没有把测试框架作为代码库的一部分，如果我用了，别人拿过去，就没法编译。

这个东东需要具备哪些行为呢？无非就是增加和查找。对一个URL，将会在用户自定义数据库和系统数据库中依次查找，查找的结果，要么切断，要么通过。最初，我们定义了一个枚举类型来代表这种结果：

    // representing the action we'll execute according to this url
    enum UrlAction
    {
        ACTION_CUT, // can not access this page
        ACTION_PASS, // allow
        NOT_FOUND // haven't found in database or cache
    };

第一次看到这个枚举时，我隐约觉得有点不安，也许这就是Martin Fowler所说的“臭味”吧，但又不知问题具体是什么。后面，我们就会发现它对重构过程所造成的障碍。

现在开始写测试：

首先，如果添加了一个条目，这个条目必须在里面：

    void UrlCacheTest::testAddedItemMustExists()
    {
        UrlCache cache;
        cache.add("www.163.com", ACTION_CUT);
        assert(cache.check("www.163.com") == ACTION_CUT);
        cache.add("www.google.com", ACTION_PASS);
        assert(cache.check("www.google.com") == ACTION_PASS);
    }

第二，如果Cache里没有某个条目，查找时当然就不能有。这个测试真是有点小儿科。但我就曾经犯过低级错误，在函数里返回一个固定值，然后就因为别的事，把这段代码给忘掉了。还是小心为上。考虑一个东西需不需要测试，是一个折衷的过程，如果你不像我这般粗心，就可以少写点代码。而我倾向于尽量多加一些防护措施。

    void UrlCacheTest::testNonexistingItem()
    {
        UrlCache cache;
        assert(cache.check("www.163.com") == NOT_FOUND);
    }

第三，如果Cache的空间已经满了，再增加一条Url时，就会按LRU算法淘汰出一个候选者。这个淘汰算法对Cache的效率至关重要，而且也是比较复杂的部分，因此要重点测试。

乍一看不太好办，不深入其实现机制，如何能知道谁被淘汰了呢？不过此处可以用一点小技巧。如果我们让这个Cache只有两个元素的空间，在已经加过两条Url之后，再增加一条，必须会有一条被淘汰出来，我们可以通过check来判断淘汰的对象是不是我们所希望的。

同时，这也给我揭示了一个未曾想到的问题：UrlCache的大小虽然固定，但必须在建立时指定。因为我们需要依据命中率，调整应该使用的Cache大小。因此，写出测试代码如下：

    void UrlCacheTest::testLRUWithNotEqualHits()
    {
        UrlCache<2> cache;
        cache.add("www.163.com", ACTION_CUT);
        cache.add("www.google.com", ACTION_PASS);
        cache.check("www.163.com"); // 163.com现在两次命中
        cache.add("www.sina.com", ACTION_PASS);
        // cache中只有两个空间，必须替换掉一个
        // google此时应该被替换掉，因为它比16少一次命中。
        assert(cache.check("www.google.com") == NOT_FOUND);
    }

UrlCache的大小为何用模板，而不在构造函数中传进去呢？主要考虑是这样实现起来最简单，可以用数组作为存储容器，很自然地就满足了固定大小的要求。当然，如果这个Cache需要动态调整大小，那就只能通过构造函数传了。

加了模板参数后，前两个测试的代码都得略微调整一下。

还有一个比较细节的问题：连续两次增加同一个URL，却使用不同的UrlAction，这个Cache应该是什么行为？是抛出异常、返回错误值、还是忽略？无论如何，都必须有确定的行为。最简单的办法就是只保留最后一次的结果。

    void UrlCacheTest::testAddTwice()
    {
        UrlCache<2> cache;
        cache.add("www.163.com", ACTION_CUT);
        cache.add("www.163.com", ACTION_PASS);
        // 只保留最后一次的结果
        assert(cache.check("www.163.com") == ACTION_PASS);
    }

现在，写一个do-nothing的UrlCache，让整个程序通过编译吧：

    template<int SIZE>
    class UrlCache
    {
    public:
        UrlCache();
        
        void add(const ::std::string& url, UrlAction action);
        UrlAction check(const ::std::string& url) const;
    private:
        // not implemented
        UrlCache(const UrlCache&);
        UrlCache& operator=(const UrlCache&);
    };

    template<int SIZE>
    UrlCache<SIZE>::UrlCache()
    {
    }

    template<int SIZE>
    void UrlCache<SIZE>::add(const ::std::string& url, UrlAction action)
    {
    }

    template<int SIZE>
    UrlAction UrlCache<SIZE>::check(const ::std::string& url) const
    {
        return NOT_FOUND;
    }

最后，加上一个所有测试的驱动函数，以及整个测试程序的入口：

    void UrlCacheTest::testAll()
    {
        testAddedItemMustExists();
        testNonexistingItem();
        testLRUWithNotEqualHits();
    }
    int main(int argc, char* argv[])
    {
        UrlCacheTest tester;
        tester.testAll();
        return 0;
    }

编译，运行，不出所料，在第一个assert的地方就出错了。

剩下的任务，就是让测试通过。

用一个叫UrlItem的类来表示Cache中的一个元素：

    class UrlItem 
    {
    public:
        UrlItem();
        UrlItem(const ::std::string& url, UrlAction action);
        const ::std::string& url() const;
        UrlAction action() const;
    private:
        ::std::string url_;
        UrlAction action_;
    };

我决定用一个固定的数组作为容器，用一个int型的值来代表该UrlItem当前命中的情况：

    template<int SIZE>
    class UrlCache
    {
        // …
    private:
        ::std::pair<UrlItem, int> items_[SIZE];
    };

然后，实现两个成员函数：

    template<int SIZE>
    void UrlCache<SIZE>::add(const std::string& url, UrlAction action)
    {
        int hits = items_[0].second;
        int candidate = 0;
        // loop begins from 1, because item 0 is the default value
        for (int i = 1; i < SIZE; ++i)
        {
            if (items_[i].second < hits)
            {
                hits = items_[i].second;
                candidate = i;
            }
        }
        items_[candidate] = std::make_pair(UrlItem(url, action), 1);
    }

    template<int SIZE>
    UrlAction UrlCache<SIZE>::check(const std::string& url)
    {
        for (int i = 0; i < SIZE; ++i)
        {
            if (items_[i].first.url() == url)
            {
                items_[i].second++;
                return items_[i].first.action();
            }
        }
        return NOT_FOUND;
    }

在实现这两个成员函数的过程中，我查了查LRU(最近最少使用)算法和它的几个变种，仔细研读，因为此前我从来没有做过这样的东西。我选择了最简单的一种方法，而且还不敢保证我是否理解有误，或者实现上有问题。不过我还是比较有信心，因为我有安全网———单元测试。

大概花了10分钟时间，东调调，西整整，终于把编译和测试都通过了。这就是我们需要的功能，不多不少，刚好够用。眼睛有点酸，起身接了点水，一边喝着，一边眺望远方，顺便活动一下脸部神经。

这个Cache工作得很好。通过了单元测试，在命中率测试中也获得了高分。我有点自鸣得意，把代码和数据都提交到代码库中，然后带着愉快的心情，转而编写程序的其它部分。

过了不久，我又碰到另一部分代码，需要根据IP地址，反向查找对应的主机名。这个过程需要联络DNS服务器，这个过程受网络和DNS响应速度的影响很大，延迟和不确定性更胜过查找URL数据库，因而我需要将取得的IP/主机名对应关系也记住。这可以用一个map来实现，但我们的程序需要无人看守地不间断运行，如果任由这个map不断增大，物理内存到最后就会因为这个map而耗光。于是我又想到了曾经派上过用场的Cache。可原来的UrlCache完全是为UrlItem量身定制，要想重用，有点难度。我开始埋怨自己当初想得不够周全了。埋怨归埋怨，问题还得解决。我可不想再重新写一个Cache，重构的时机到了。

###重构---将元素类型参数化

工作量不小，悠着点儿，一步一步来。

首先把UrlCache改为Cache，以符合其作为通用Cache的功能。同时，将头文件，实现文件、工程文件和Makefile都相应地更新。

然后，在测试程序中，把所有引用UrlCache的代码全部换成Cache。这一步倒是可以借助于文本替换工具。

编译，让编译器来帮我们查找没有发现的引用点。

下一步，考虑替换UrlItem。先思考一下：从更抽象的层次考虑，一个Cache，有哪几个重要的概念？

  + 键

  + 对应的值

  + 是否命中

在前一个版本中，我把后两个概念混在一起，在UrlAction中加了一个NOT_FOUND，原来这就是我觉得不对劲的地方。因为命中与否是Cache本身的概念，而键和值则随Cache元素类型而变化。但犯错误终归不可避免，现在就得纠正这个错误，把这两个概念分开。

因此，首先重构check成员函数，在UrlAction中去掉NOT_FOUND

    enum UrlAction { ACTION_CUT, ACTION_PASS };

一鼓作气，让我们把check成员函数的声明改为：

    bool check(const std::string& url, UrlAction& action);

如果Url在Cache中存在，则返回true，并将对应的值拷贝到输出参数action中，否则，返回false，不改变参数action的内容。

在测试程序中把所有对check的调用替换成新版本，然后，编译并运行。一切OK。

接口已经确定下来，现在就该大刀阔斧地修改实现代码了。这中间，没有遵循什么固定的章法，因为接口已经确定，函数体可以一个一个地改。最终的结果就是：类模板Cache具有三个模板参数：键类型、值类型和大小。最终的代码是这样：

    #include <utility>

    template<typename Key, typename Value>
    class CacheItem
    {
    public:
        typedef Key key_type;
        typedef Value value_type;

        CacheItem() {};
        CacheItem(const Key& key, const Value& value);
        const Key& key() const { return assoc_.first; }
        const Value& value() const { return assoc_.second; }
        int hits() const { return hits_; }
        void incHits() { hits_++; }
    private:
        :: std::pair<Key, Value> assoc_;
        int hits_;
    };

    template<typename Key, typename Value>
    CacheItem<Key, Value>::CacheItem(const Key& key, const Value& value)
    : assoc_(std::pair<Key, Value>(key, value)), hits_(0)
    {
    }

    template<typename Key, typename Value, int SIZE>
    class Cache 
    {
    public:
        Cache();
        void add(const Key& key, const Value& value);
        bool check(const Key& key, Value& value);
    private:
        // not implemented
        Cache(const Cache&);
        Cache& operator=(const Cache&);

        CacheItem<Key, Value> items_[SIZE];
    };

    template<typename Key, typename Value, int SIZE>
    Cache<Key, Value, SIZE>::Cache()
    {
    }

    template<typename Key, typename Value, int SIZE>
    void Cache<Key, Value, SIZE>::add(const Key& key, const Value& value)
    {
        int hits = items_[0].hits();
        int candidate = 0;
        for (int i = 0; i < SIZE; ++i)
        {
            if (items_[i].hits() < hits)
            {
                hits = items_[i].hits();
                candidate = i;
            }
        }
        items_[candidate] = CacheItem<Key, Value>(key, value);
    }

    template<typename Key, typename Value, int SIZE>
    bool Cache<Key, Value, SIZE>::check(const Key& key, Value& value)
    {
        for (int i = 0; i < SIZE; ++i)
        {
            if (items_[i].key() == key)
            {    
                items_[i].incHits();
                value = items_[i].value();
                return true;
            }
        }
        return false;
    }

因为增加了模板参数，所以原来的测试需要稍做修改。而且在网页过滤模块中已经用到了这个Cache，所以也得把对应的地方改过来，编译器可以在我们疏忽时提醒我们。

运行测试，有一个断言居然出错了。哦，原来是我图方便，在测试代码中直接拷贝、粘贴造成的。这让我又一次感激这些不起眼的测试代码。

既然这个Cache还要用于IP/主机名的缓存，那也为它增加一个测试吧。在实际中，IP地址是用一个类来表示的，不过这里，我们还是直接用点分十进制格式的字符串表示，别让那些无关紧要的东西污染了正题。这样，键和值都是标准string。

    void testIPHostPairCache()
    {
        Cache<std::string, std::string, 2> cache;
        std::string host;
        cache.add("10.10.10.37", "RMY");
        cache.add("10.10.10.31", "sunwang");
        // 37现在两次命中
        cache.check("10.10.10.37", host);
        cache.add("10.10.10.38", "Xiaoqi");

        // 31此时应该被替换掉，因为它比37少一次命中
        assert(!cache.check("10.10.10.31", host));
    }

好，我们的Cache又在另一个地方派上了用场，这让我觉得有点兴奋。兴许什么时候，还能够重用这段不起眼的代码呢？看来，该写个小小的Readme了。

我们的重构到此告一段落，但如果要挑刺，其实还有很多值得改进的地方，比如：

  1. Cache对键和值都要求有默认构造函数和拷贝赋值运算符，这在某些情况下不现实。

  2. 判断命中时，直接以键值是否相等作为标准，没有灵活性。假设为了避免拷贝，键值中存储的是指针，直接以指针进行比较显然是不对的，那就得对比较标准参数化，学习一下标准库，默认情况下，使用std::equal_to，同时又允许使用者提供自己的谓词。

  3. 成员函数check，最多要执行SIZE次元素的大小比较，而add最多只需要SIZE次整数比较。但实际使用时，check调用的次数远远比add多。如果有办法让check只有O(logN)的复杂度，让add的运行时间长一点也是值得的。可以用堆排序（如标准库里的堆操作）做到这一点。

但现在的两个类型，我们都已经满足，而且要增加也非难事，对既有的使用者也没有影响。所以，还是留待以后需要此功能时，再加不迟，何苦现在就去加上那或许根本用不上的功能呢？把宝贵的时间用在更有意义的地方吧。我已略感困倦，火炬大厦的钟声已然敲响，该下班了。

