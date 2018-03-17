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