# 类型系统的基本概念


Type system的基本目标是保证程序中不会出现某些错误，本文先谈什么叫错误，错误都有哪些种类，然后再谈类型系统的一些基础概念。


## execution error {#execution-error}

需要区分两种情况：trapped errors和untrapped
errors，前者发生时会导致程序立即终止，而后者发生时不会导致立即终止，而是继续执行但后续行为不可预知。典型的untrapped
error如C语言里面的数据越界读写。


## language safety {#language-safety}

如果一段程序不会发生untrapped
error，就称这段程序是safe的。如果一个语言写成的程序都是safe的，那么就称该语言为safe
language；否则就是unsafe language。


## Well-behaved vs. ill-behaved {#well-behaved-vs-dot-ill-behaved}

设计语言时会指定所有可能发生的execution error的一个子集，为forbidden
error。这个子集必须包含所有的untrapped error和一部分trapped
error。如果一段程序不会出现forbidden
error，就称之为well-behaved，否则称为ill-behaved。

由此定义可知，well
behavior比safety要求更严格。一段safe的程序并不一定是well-behaved，因为还可能发生trapped
error。但是如果是well-behaved，那么一定safe。


## type system {#type-system}

排除程序中错误的手段有多种，type system只是其中一种。TAPL中对type
system的定义如下：

> A type system is a tractable syntactic method for proving the absence of
> certain program behaviors by classifying phrases according to the kinds
> of values they compute.

根据以上定义，做三点说明：

1.  是用来对系统进行推理的一种工具
2.  其目标是排除程序中某些特定类型的错误，而排除哪些错误与与具体类型系统的具体目标有关
3.  其手段是对程序中变量的取值集合进行分类，相当于对程序动态行为的一种静态近似


## type {#type}

type定义了变量可能的取值集合以及允许在其上执行的操作集合。如：整数类型，浮点数类型，布尔类型。


## typed languages vs. untyped languages {#typed-languages-vs-dot-untyped-languages}

配备type system的语言叫typed language，没有配备type
system的语言叫untyped language。Pure lambda calculus就是典型的untyped
language。


## type checking {#type-checking}

一般来说，type system的基本目标是要保证safety，进阶目标是要保证well
behaviour，这因语言的具体设计而异。保证type system实现其目标的手段type
checking，这是在运行前对程序进行检查，由于是在运行前检查，也叫static
checking，执行检查的工具是type checker。如果一个程序经受住了type
checker的检查，则称之为well typed，否则称为ill typed。

保证程序不发生错误的手段不是只有type checking，比如untyped
language（如LISP）根本没有类型系统，但它可以使用运行时检查来排除掉所有的forbidden
error。由于是在运行时检查的，所以叫dynamic checking。


## Strongly checked vs. weakly checked {#strongly-checked-vs-dot-weakly-checked}

现实中一些语言并不保证safety或well
behaviour，也就是说它们并不保证检查出所有的untrapped
error，我们称这类语言为weakly
checked，而称提供了这种保证的语言为strongly checked。


---

> 作者: [lazypanda](https://github.com/wanghuibin0)  
> URL: https://simplecoding.fun/posts/2022/11/type-system/  

