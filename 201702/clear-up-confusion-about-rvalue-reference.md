# 一些对右值引用的误解

右值引用比较早就接触了，但是一直以来主动使用比较少。今天又看了一下《Effective Modern C++》Chapter 5，突然对`std::forward`又有一些疑惑。

## 代码示例
摘抄自书里的代码段小做修改
```cpp
#include <iostream>
#include <utility>
using namespace std;

//void process(const int& lvalArg) {
//    cout << "This is lvalArg: " << lvalArg << endl;
//}

void process(int&& rvalArg) {
    cout << "This is rvalArg: " << rvalArg << endl;
}

template <typename T>
void coutAndProcess(T&& param) {
    cout << "Logging..." << endl;
    process(param);
}

int main()
{
    int i = 10;
    coutAndProcess(std::move(i));
}
```

这里我注释掉了`void process(const int& lvalArg)`，导致`process(param)`会编译不通过，提示没有匹配的函数声明

## 疑惑

param是一个Meyers说的*universal reference*或者cppreference指的[Forwarding references](http://en.cppreference.com/w/cpp/language/reference#Forwarding_references)，当我用`coutAndProcess(std::move(i))`调用时，`param`是一个右值引用(rvalue-reference)

此时为什么`process(param)`会失败呢？我不是传递了一个右值引用给参数列表为一个右值引用的`void process(int&& rvalArg)`吗？

而如果我把调用改为`process(std::forward<int>(param))`就可以编译通过，然而`std::forward`的返回式是`int&&`，同样是一个右值引用，为什么这样就是合法的呢？

## 答案

答案就在于，其实我陷入了一个思维误区

首先，一个函数的参数列表(**parameter** list)和传给它的参数列表(**argument** list)本质上并不是相同的含义。中间的含义在于**传递**

对于简单的函数声明这可能是不易发现的，比如：
```cpp
void foo(int v) {
    cout << "I got: " << v << endl;
}

int i = 10;
foo(i);
```
parameter v是个int，argument i也是个int，没有任何问题，实际上这是一个**按值传递**的含义

如果我把foo改成这样：
```cpp
void foo(int& v) {
    ...
}
```
那这就是一个**按引用传递**，parameter `v`是对argument `i`的引用，这是语言层面的，底层则是指针关联。

废话这么多的原因在于，我的疑惑源于像parameter是`int&&`的函数传递一个`int&&`类型的argument应该是合法的，其实这是非法的，因为**rvalue-reference本身是一个lvalue，这是它的变量属性，而rvalue-reference是它的变量类型**。
我们都知道，不能向右值引用变量传递一个左值，即右值引用不能绑定左值，正如左值引用不能绑定一个右值一样。所以向parameter是`int&&`的函数传递一个`int&&`类型的左值argument自然就是非法的。

那么`std::forward<int>(param)`呢？

答案也很简单，要知道`std::forward`只不过是一个函数模板，它被实例化成一个函数，该函数被调用返回一个右值引用（这块不清楚的话可以查阅一下《Effective Modern C++》Chapter 5），这个被返回的右值引用**是一个右值**，将其传递给parameter是`int&&`的函数自然也是合法的。

> Rvalue references returned from functions are defined to be rvalues

### 如果返回的是左值引用呢？
于此同时我也做了另外一个小实验：
```cpp
int& someFunc(int &v);
process(someFunc(param));   // Compile Error
```
一个被返回的左值引用是一个左值，而非右值。这与一个被返回的右值引用是不同的。
所以你可以这样：`someFunc(param) = 11;`，却不可以这样`std::forward(param) = 11`

BWT，对于返回一个值而非引用的函数，其返回值也是一个右值，当然这个是很好理解的。
