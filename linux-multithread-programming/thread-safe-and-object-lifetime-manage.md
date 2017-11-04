# 线程安全和对象生命期管理

主要谈论三个话题：
1. 线程安全
2. 对象创建做到线程安全
3. 对象析构做到线程安全

## 线程安全

线程安全比较重要的一个概念是：**不需要调用方额外的同步和加锁动作也能在多线程环境下正常表现**
主要利用两个工具类 `MutexLock` 和 `MutexLockGuard`，前者是封装临界区，后者负责加解锁

## 对象构造

对象构造的线程安全是简单的，毕竟对象都每不存在，多线程场景也比较少
唯一的要求是：**不要泄露`this`指针**

需要强调的是：到最后一行代码也不要泄露，因为类有可能是父类
假设我们有个类Foo，它有一个子类Bar。如果在Foo类中泄露this指针，比如通过观察者模式注册出去，当我们在构造子类Bar时就比较危险了，如果Bar构造失败，而又通过其父类Foo暴露了this指针，导致难以预料的运行结果

如果真的要暴露this指针，也要等到对象构造完成。也就是二段式构造 —— 构造函数 + `initialize()`

## 对象析构
书上例举了几个对象析构的竞态条件：

> 1. 在即将析构一个对象时，从何而知此刻是否有别的线程正在执行该对象的成员函数?
> 2. 如何保证在执行成员函数期间，对象不会在另一个线程被析构?
> 3. 在调用某个对象的成员函数之前，如何得知这个对象还活着?它的析构函数会不会碰巧执行到一半?

原则就是在多线程场景，使用`share_ptr`来一劳永逸地解决问题

回顾上面提到的线程安全条件：不需要调用方额外的同步和加锁动作也能在多线程环境下正常表现。如果我们要实现线程安全的析构，那就要在析构内部封装临界区，比如：

```cpp
Foo::~Foo()
{
    MutexLockGuard lock(mutex_);
    // free internal state
}

Foo::update()
{
    MutexLockGuard lock(mutex_);
    // update internal state
}

extern Foo* x ; // visible by all thread

// thread A
delete x;
x = NULL;

// thread B
if (x) {
    x->update();
}
```

最大的问题在于：`mutex_`作为成员变量会被析构掉，而其他调用点并不知道，导致不可知的后果
另外`x=NULL;`只能用来在**单线程**场景下防止野指针和多次释放的问题，但是在**多线程场景下是无效的**

## shared_ptr 和 weak_ptr

`shared_ptr` 控制对象的生命期
`weak_ptr` 不控制对象的生命期，但可以通过`lock()`提升为`shared_ptr`

`shared_ptr`/`weak_ptr`的线程安全级别和STL中的容器一样。但是注意，`weak_ptr`的`lock()`

使用`shared_ptr`要**防止引用循环**，所以通常是owner持有指向child的`shared_ptr`，child持有指向owner的`weak_ptr`。
用STL容器维护的时候，可以采用weak_ptr

## shared_ptr线程安全
- 一个shared_ptr可以被多个线程同时读取
- 一个shared_ptr被多个线程写入时（包括赋值，reset析构），需要加锁保护

*注意：这里是shared_ptr对象本身的线程安全，而不是其管理的对象的线程安全*

一般对**多个线程可访问的**shared_ptr的读写操作，使用mutex互斥锁保护。采用**Local Copy**的方法，在临界区外只操作局部shared_ptr，这样可以充分缩小临界区

```
MutexLock mutex;
shared_ptr<Foo> globalPtr;

void read()
{
    shared_ptr<Foo> localPtr;
    
    {
        MutextLockGuard lock(mutex);
        localPtr = globalPtr;   // globalPtr被读取，通过mutex保护
    }

    doSomething(localPtr);
}

void write()
{
    shared_ptr<Foo> newPtr;

    {
        MutexLockGuard lock(mutex);
        globalPtr = newPtr;     // globalPtr被赋值，通过mutex保护
    }

    doSomething(newPtr);
}
```

如果需要销毁对象，可以这么做
```
void deleteObj
{
    shared_ptr<Foo> localPtr;

    {
        MutexLockGuard lock(mutex);
        // globalPtr本身已被清空，通过mutex保护，只不过对象还没被析构
        globalPtr.swap(localPtr);   
    }

    localPtr.reset();   // 通过局部shared_ptr来析构对象
}
```
## shared_ptr技术与陷阱

### 避免意外延长对象生命期
例如在用STL容器维护的时候，可以采用weak_ptr，如下对象池案例，需求是池里没有的时候创建，外部没有引用的时候销毁

```
class StockFactory: boost::noncopyable
{
    public:
        shared_ptr<Stock> get(const string& key);

    private:
        mutable MutexLock mutex_;
        map<string, weak_ptr<Stock>> stocks_;
}
```
由于weak_ptr不控制生命期，所以不会因为`StockFactory`的较长生命期影响对象的生命期，只需要在 get 的时候将`weak_ptr`提升为一个局部`shared_ptr`变量并作为返回即可

### 析构动作在创建时指定

> 这是一个非常有用的特性

比如在对象池里面，指定shared_ptr的deleter，在对象析构时调用deleter，在deleter中对 `stocks_` 相应的key做erase，这样就算彻底**打扫干净战场**了

### pass by const reference
在需要作为函数参数时，多数情况下，只需要在最外层维持一个shared_ptr，之后的函数传递都使用pass by const reference，就可以避免传参的时候产生复制和修改引用计数

### shared_from_this
所谓 shared_from_this，就是在**要暴露this指针给外部**时（比如注册回调，就像上面说到的deleter），为了保证：
1. 回调发生时this所指对象还存在，而不至于因为对象已析构导致core dump
2. this指针不会被随便操作，比如被delete this
所以改用暴露shared_ptr给外部

但是，我们并不能直接这样：`shared_ptr<MyClass> localPtr(this); return localPtr;`，因为像这样`ptr1`和`ptr2`不是共享计数的：
```
MyClass *obj = new MyClass();
shared_ptr<MyClass> ptr1(obj);
shared_ptr<MyClass> ptt2(obj);
```
同样的道理对`this`指针也适用，因为不是共享计数，则这多个shared_ptr析构的时候会调用多次`delete this`
