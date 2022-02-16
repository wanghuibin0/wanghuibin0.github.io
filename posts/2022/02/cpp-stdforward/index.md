# 完美转发与std::forward


## 问题 {#问题}

一个模板函数f调用另一个函数g时，要将自己的参数转发给g，在转发过程中，有时我们想要保持被转发参数的类型不变，包括是否是const，是左值引用还是右值引用。


## 解决方案 {#解决方案}

定义模板函数f时，将其参数定义为模板类型参数的右值引用，然后针对需要需要转发的参数，通过调用std::forward转发给g

```c++
template <typename F, typename T1, typename T2>
void flip(F f, T1 &&t1, T2 &&t2)
{
    f(std::forward<T2>(t2), std::forward<T1>(t1));
}
```


## std::forward是怎么实现的 {#std-forward是怎么实现的}

1.  实现方式1

    ```c++
    template<typename T>                // For lvalues (T is T&),
    T&& std::forward(T&& param)         // take/return lvalue refs.
    {                                   // For rvalues (T is T),
        return static_cast<T&&>(param); // take/return rvalue refs.
    }
    ```
2.  实现方式2

    ```c++
    template<typename T>
    struct identity {
        typedef T type;
    };
    template<typename T>
    T&& forward(typename identity<T>::type&& param)
    {
        return static_cast<identity<T>::type&&>(param);
    }
    ```

最好是使用第二种方式，因为用第一种的话可以写std::forward(x)，这样返回的总是左值引用。
而第二种方式禁止了自动推导，必须使用std::forward&lt;T&gt;(x)这种方式调用。

