# 线程管理
用std::thread launch 一个线程, 在std::thread对象被销毁之前, 一定要调用`detach`或者`join`, 决定是让线程自己运行或者等待线程运行完成
`detach` 一个 std::thread, 则线程可以在 std::thread 对象被销毁后继续运行, 所以要注意不要在线程函数中引用局部变量
`join` 一个 std::thread 会等待该线程执行完成, 之后该std::thread就不再和任何线程关联(底层资源也会被清理), 所以只能被调用一次

如果需要detach线程, 一般在生成std::thread后就调用detach了. 但是从生成std::thread到join它之间可能会发生异常(主线程需要去做别的事情), 这时候可以用`thread_guard`以RAII的手法, 保证std::thread一定能被join

```cpp
class thread_guard
{
    std::thread& t_;
public:
    explicit thread_guard(std::thread& t): t_(t) {}
    ~thread_guard()
    {
        if (t_.joinable())
        {
            t_.join();
        }
    }
    thread_guard(thread_guard const &) = delete;
    thread_guard& operator=(thread_guard const&) = delete;
};
```

thread id 怎么和调试时关联?

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

# 避免死锁
避免死锁的常见的建议是永远采用相同的顺序lock多个mutex, 采用 `std::lock` 可以原子地锁定多个mutex, 避免死锁, 同时通过 `std::lock_guard<std::mutex>(some_mutex, std::adopt_lock)` 以RAII的方式释放锁, 这里`std::adopt_lock`表示lock_guard只是接管这个mutex而不是尝试lock它, 因为已经被 `std::lock` 锁定过了

`std::lock` 保证多个mutex能够同时被锁定, 而不是分步骤锁定, 后者就算采用相同顺序锁定mutex, 也可能导致死锁, 比如带锁的swap

相比`std::lock_guard`, `std::unique_lock`可以支持锁传递, 以及配合 `std::defer_lock` 延迟锁定mutex直到主动调用`lock()`, 但是由于需要标记是否已经锁定了相应的mutex, 所以相比`std::lock_guard`消耗多点内存和效率

# 保护共享数据的初始化
Double Check lock 存在data race condition, 两种办法实现线程安全的共享资源初始化
1. 采用`std::call_once`配合`std::once_flag`, 保证某个函数只调用一次. 类似于`pthread_once`和`pthread_once_t`
2. 如果是要实现全局唯一的单例(只能初始化一次), 可以采用static的实例变量, 但是这个必须在c++11之后的编译器才能支持线程安全

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

陈硕在他的书<Linux多线程服务端编程>中提出另外一种方法, 他称作"copy on write", 结合`std::shared_ptr`来实现, 在写时复制一份同时reset shared_ptr, 这样就不会影响正在读取共享数据的读者. 只是读者的数据会稍微落后, 但是一般在网络环境下都没什么大问题.

```cpp
#include <mutex>

shared_ptr<std::map<std::string, int>> data; //用shared_ptr管理共享数据
std::mutex data_mutex;

int FindData(const std::string& key)
{
    std::shared_ptr local_data;     //本地Shared_ptr变量

    {
        // 锁保护 data, 同时共享到local_data, 增加引用计数
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

    if (data.unique())
    {
        std::lock_guard lock(data_mutex);

    }
}
```

#条件变量
`std::condition_variable`是标准库提供的条件变量, 主要的方法有 `wait()`, `notify_one()`, `notify_all()`. 必须配合`std::unique_lock`使用, 因为`wait()`的时候需要有`unlock`和`lock`的操作, 这种灵活性是`std::lock_guard`不能提供的. 

使用条件变量要注意 "spurious wakeup", 例如使用pthread API时
```cpp
std::queue<int> data_queue;
pthread_mutex_t mutex;
pthread_cond_t cond;

// 没有处理函数异常
void thread()
{
    pthread_mutex_lock(&mutex);
    // 用循环检查确保条件真实被满足了
    while(data_queue.empty())
    {
        pthread_cond_wait(&cond, &mutex);
    }
    // use data ...
}
```

`std::condition_variable`提供了一种更简单的方法, 配合lambda语法, 可以简化代码

```cpp
std::queue<int> data_queue;
std::mutex data_mutex;
std::condition_variable cond;
void thread()
{
    // 必须用 std::unique_lock
    std::unique_lock<std::mutex> lock(data_mutex);
    cond.wait(lock, []{return !data_queue.empty();});

    /*
    * 也可以这样:
    * while(data_queue.empty())
    * {
    *     cond.wait(lock);   
    * }
    */
}
```

条件变量适合那种多次条件满足后notify的情况, 但如果是一次性的通知, 可以用 future

#Future编程
标准库提供了两种future封装: `std::future` 和 `std::shared_future`
