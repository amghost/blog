# 保护共享数据
使用 `std::mutex` 和 `std::lock_guard`
```cpp
std::list<int> some_list;
std::mutex some_mutex;
void add_to_list(int a)
{
    std::lock_guard<std::mutex> guard(some_mutex);
    some_list.push_back(a);
}
```

# 使用锁的灵活性
相比`std::lock_guard`, `std::unique_lock`可以支持锁传递, 以及配合 `std::defer_lock` 延迟锁定mutex直到主动调用`lock()`, 但是由于需要标记是否已经锁定了相应的mutex, 所以相比`std::lock_guard`消耗多点内存和效率, 所以如非必须, 建议都使用`std::lock_guard`. 只有在以下需求的时候使用 `std::unique_lock`:
1. 需要延迟加锁
2. 需要锁传递

# 保护共享数据的初始化
有时候我们需要做 Lazy Initialization(比如网络连接, 或者大内存资源), 在资源需要使用到时再初始化它. 在非多线程时我们大概会这样(伪代码)
```
If Resource Is Not Initialized:
    Initialized The Resource

Use the Resource
```

改为多线程环境下, Double Check lock 存在data race condition, 两种办法实现线程安全的共享资源初始化
1. 采用`std::call_once`配合`std::once_flag`, 保证某个函数只调用一次. 类似于`pthread_once`和`pthread_once_t`
2. 如果是要实现全局唯一的单例(只能初始化一次), 可以采用static的实例变量, 但是这个必须在c++11之后的编译器才能支持线程安全, 否则会有线程安全问题

```cpp
std::shared_ptr<some_resource> resource_ptr;
std::once_flag resource_flag;

void init_resource()
{
    resource_ptr.reset(new some_resource);
}

void foo()
{
    std::call_once(resource_flag, init_resource);
    resource_ptr->do_something();
}
```


个人看法: 如果是要实现线程安全的单例, 且编译器满足要求(支持c++11), 则static实例变量足够. 但是如果是存在可能要多次初始化的过程, 比如某个全局数据在初始化之后经过一段时间要置为无效, 下次还有初始化的需求; 或者存在多个地方可以初始化的时候, 总之就是不属于要实现单例模式时, 就要使用`std::call_once`和`std::once_flag`. 而如果并不支持c++11, 那就还要用`pthread_once`和`pthread_once_t`

# 保护读多写少的共享数据
一种办法是使用读写锁, c++标准库没有现成的, 可以使用boost的`boost::shared_mutex`和`boost::shared_lock`

```cpp
#include <boost/thread/shared_mutex.hpp>
#include <mutex>

boost::shared_mutex mutex;
std::map<std::string, int> data;

int FindData(const std::string& key)
{
    // 读锁, 使用 boost::shread_lock
    boost::shared_lock<boost::shared_mutex> lock(mutex);
    const auto it = data.find(key);
    return (it == data.end()) ? 0 : it->second;
}

int UpdateData(const std::string& key, int new_value)
{
    // 写锁, 使用 std::lock_guard
    std::lock_guard(boost::shared_mutex) lock(mutex);
    data[key] = new_value;
}
```

分读写锁相比读写使用同一个普通的锁, 好处在于读者之间可以不互斥, 并发度可以提高. 但是局限在于, 写者不仅与其他写者互斥, 同时还与当前持有读锁的读者互斥, 必须等到其他读者的读操作完成才能获取到锁进行修改.

陈硕在他的书<Linux多线程服务端编程>中(第53页)提出另外一种方法, 他称作"copy on write", 结合`std::shared_ptr`来实现, 在写时复制一份同时reset shared_ptr, 这样就不会影响正在读取共享数据的读者. 只是读者的数据会稍微落后, 但是一般在网络环境下都没什么大问题.

```cpp
#include <mutex>

shared_ptr<std::map<std::string, int>> data; //用shared_ptr管理共享数据
std::mutex data_mutex;

int FindData(const std::string& key)
{
    std::shared_ptr local_data;     //本地Shared_ptr变量

    // 锁保护 data, 同时共享到local_data(这里没有复制), 增加引用计数
    {
        std::lock_guard lock(data_mutex);
        local_data = data;
    }

    // 现在可以安全地访问local_data指向的数据了
    const auto it = local_data->find(key);
    return (it == local_data->.end()) ? 0 : it->second;
}

int UpdateData(const std::string& key, int new_value)
{
    std::lock_guard lock(data_mutex);

    if (!data.unique())
    {
        data.reset(new std::map<std::string, int>(*data));
    }

    assert(data.unique());
    (*data)[key] = new_value;
}
```