# 笔记-LLVM IR中的undef和poison


1.  undef用来表示该常量有可能取任意值，这有利于编译器做优化，因为编译器可以用任意值来替代undef。

    需要注意的是，undef的取值可以在其活跃范围内随时变化，这会带来一些违反直觉的结果。比如：
    ```llvm
    %A = undef
    %B = xor %A, %A
    ```
    与之等价的是 `%B = undef` ，而不是直观上的 `%B = 0` 。

2.  poison用来表示错误操作的结果。

    典型的例子是，带有nsw/nuw flag的add指令，如果结果溢出，就会产生poison结果。

    对于大多数指令（除了select），如果有一个操作数是poison，那么其结果也是poison。

3.  与undef和poison密切相关的一条指令是freeze，它是用来停止undef/poison的传播的。

    如果其操作数是undef/poison，那么其结果就变成了固定值（具体是多少是任意取的）；如果是其他操作数，则相当于一条nop。比如：
    ```llvm
    %A = undef
    %B = freeze %A
    %C = xor %B, %B
    ```
    这段代码的执行结果会是： `%C = 0` 。

4.  从语义角度看，poison是比undef更「强」的概念。

    在能使用poison的地方，用undef来替代是安全的，反之则不然。

    而从有利于优化的角度来看，应该尽可能使用poison，因为这相当于让编译器看到了更「具体」的信息，可以做更激进的优化。


## refs {#refs}

-   &lt;https://llvm.org/docs/LangRef.html#undefined-values&gt;


---

> 作者: [lazypanda](https://github.com/wanghuibin0)  
> URL: https://simplecoding.fun/posts/2024/02/llvm-undef-poison/  

