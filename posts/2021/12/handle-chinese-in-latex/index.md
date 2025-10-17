# Latex排版中文的问题汇总


-   Latex排版中文的问题汇总 涉及两方面：
-   对中文字体的支持
-   对中文排版细节的处理

几个相关的宏包： - CJK宏包：提供了对中文字体的支持 -
xeCJK和luatexja宏包：处理中文排版细节 - ctex 宏包和文档类：进一步封装了
CJK、xeCJK、luatexja
等宏包，使得用户在排版中文时不用再考虑排版引擎等细节。能够识别操作系统和
TEX 发行版中安装的中文字体。 - ctex 宏包：用于配合各种文档类排版中文 -
ctex 文档类：对LATEX
的标准文档类进行了封装，对一些排版根据中文排版习惯做了调整，包括
ctexart/ctexrep/ctexbook/ctexbeamer。

在archlinux上，安装texlive-langchinese 即可安装ctex宏包。

注意： 1. 源文件要保存为UTF-8编码。 2. 使用 xelatex 或 lualatex
命令编译。


## 例子 {#例子}

```latex
\documentclass{ctexart}
\begin{document}
在\LaTeX{}中排版中文。
汉字和English单词混排，通常不需要在中英文之间添加额外的空格。
当然，为了代码的可读性，加上汉字和 English 之间的空格也无妨。
汉字换行时不会引入多余的空格。
\end{document}
```

用xelatex编译。

```bash
xelatex test-zh.tex
```

如果是markdown或org mode文件，用pandoc转。

```bash
pandoc test-zh.tex -o test-zh.pdf --pdf-engine=xelatex -V CJKmainfont="WenQuanYi Zen Hei"
```


## refs: {#refs}

-   <https://wiki.archlinux.org/title/TeX_Live_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)>
-   一份（不太）简短的 LATEX2ε 介绍


---

> 作者: [lazypanda](https://github.com/wanghuibin0)  
> URL: https://simplecoding.fun/posts/2021/12/handle-chinese-in-latex/  

