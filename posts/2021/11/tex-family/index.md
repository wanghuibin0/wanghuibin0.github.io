# TeX Family


## TeX {#tex}

TeX 是 Donald Knuth 编写的一套排版系统，尤其擅长排版复杂的数学公式。TeX
语言定义了一套原语，用于对纯文本进行格式控制；并且在这些原语的基础上，定义了一系列宏，增加了TeX
语言的易用性。TeX 编译器用于编译 TeX文件，初期生成的文件为dvi格式。


## pdfTeX, XeTeX &amp; LuaTeX {#pdftex-xetex-and-luatex}

Donald Knuth 认为 TeX 已经够用，不希望 TeX
变得太复杂，决定不再往里面添加新功能，只接受bug fix。
随着新技术的发展，以及现代化排版的需要，一些人创建了几个TeX的变体：pdfTeX,
XeTeX &amp;
LuaTeX。这些变体其实是在TeX的基础上，扩展了TeX的能力，增加了新的原语和特性。虽然这些变体都继承了TeX，但这些变体之间，相互是不兼容的。


### pdfTeX {#pdftex}

1.  提供了直接输出pdf文件的能力
2.  对TeX的排版做了一些细节上的调整


### XeTeX {#xetex}

1.  处理UTF-8编码，处理多国语言
2.  可以方便地使用OpenType字体


### LuaTeX {#luatex}

1.  源于pdfTeX
2.  支持Lua脚本语言
3.  支持UTF-8编码
4.  可以处理OpenType字体，但处理机制和XeTeX不同
5.  支持MetaPost画图语言
6.  支持通过C/C++写的.dll或.so插件来添加扩展功能


## LaTeX {#latex}

与刚讲的三个变体不同，LaTeX
并未扩展TeX的功能，而是给TeX写了一组macro，这些macro仍然是建立在TeX原语的基础之上。也就是说，LaTeX只是一批大量精心设计的macro的集合。

1.  增加了很多对排版细节的控制，如页面布局，字体等，更加易用
2.  可以方便地添加其他宏包，以便解决特定领域的排版问题


## pdfLaTeX, XeLaTeX &amp; LuaLaTeX {#pdflatex-xelatex-and-lualatex}

-   pdfLaTeX = LaTeX macro package + pdfTeX engine

-   XeLaTeX = LaTeX macro package + XeTeX engine

-   LuaLaTeX = LaTeX macro package + LuaTeX engine


---

> 作者: [lazypanda](https://github.com/wanghuibin0)  
> URL: https://simplecoding.fun/posts/2021/11/tex-family/  

