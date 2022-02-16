# 从语义的角度理解std::move


## 问题 {#问题}

所谓右值rvalue，表明这个对象是一个临时对象，它所拥有的资源可以随时被回收。
而对于左值，机器会保证它所拥有的资源在生存期内一直有效。
但有时，我们明确地知道，某一个左值在一个程序点之后，就不会再被使用了，但是它的生存期还没结束，所以机器不能随便回收利用这个左值所拥有的资源。
这会造成低效。


## std::move如何解决此问题 {#std-move如何解决此问题}

这时程序员可以显式地用std::move，将这个左值转化为右值，表示这个左值后面不会再用了，这个左值拥有的资源可以回收在利用了。
而对于接收右值或右值引用作为参数的函数来说，因为知道这是一个临时对象（即使是通过左值转化过来的），所以就可以安全的把它的资源回收再利用，从而实现移动语义。
也正因此，调用std::move之后，就不要再用原来的左值名字去访问其资源了，已经不再安全了。


## std::move是如何实现的 {#std-move是如何实现的}

```c++
template <typename T>
typename remove_reference<T>::type&& move(T&& t)
{
    return static_cast<typename remove_reference<T>::type&&>(t);
}
```

假设X是一个Class，结合reference collapse规则

1.  如果给std::move传了一个右值X a，T会被推导为X，上面代码变为
    X&amp;&amp; move(X&amp;&amp; t) { return static_cast&lt;X&amp;&amp;&gt;(t); }
2.  如果给std::move传了一个左值X a，T会被推导为X&amp;，上面代码变为
    X&amp;&amp; move(X&amp; t) { return static_cast&lt;X&amp;&amp;&gt;(t); }

刚好是我们想要的。


## 右值引用本身是右值还是左值 {#右值引用本身是右值还是左值}

本身是左值。
这会影响重载函数的匹配结果。

