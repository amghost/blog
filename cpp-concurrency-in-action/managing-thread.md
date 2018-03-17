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