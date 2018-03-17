# 避免死锁
避免死锁的常见的建议是永远采用相同的顺序lock多个mutex, 采用 `std::lock` 可以原子地锁定多个mutex, 避免死锁, 同时通过 `std::lock_guard<std::mutex>(some_mutex, std::adopt_lock)` 以RAII的方式释放锁, 这里`std::adopt_lock`表示lock_guard只是接管这个mutex而不是尝试lock它, 因为已经被 `std::lock` 锁定过了

`std::lock` 保证多个mutex能够同时被锁定, 而不是分步骤锁定, 后者就算采用相同顺序锁定mutex, 也可能导致死锁, 比如带锁的swap

class some_big_object;
void swap(some_big_object& lhs, some_big_object& rhs);

```cpp
class X
{
private:
    some_big_object some_detail_;
    std::mutex m_;
public:
    X(some_big_object const& sd): some_detail_(sd) {}

    friend void swap(X& lhs, X& rhs)
    {
        // 引用同一对象
        if (&lhs == &rhs)
        {
            return;
        }

        // 需要同时加锁
        std::lock(lhs.m_, rhs.m_);
        std::lock_guard<std::mutex> lock_a(lhs.m_, std::adopt_lock);
        std::lock_guard<std::mutex> lock_b(rhs.m_, std::adopt_lock);
        swap(lhs.some_detail_, rhs.some_detail_);
    }
}
```
