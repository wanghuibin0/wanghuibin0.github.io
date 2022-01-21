# LaTeX如何将两段代码并排放置


1.  排版一段代码用lstlisting环境（listing宏包）
2.  将每一段代码用minipage环境包起来成为一个个box（注意设置每个minipage的宽度）
3.  中间可以用\rule命令画线，用\hspace调整间距
4.  若要给代码整体画个外框，可以将所有box一起放在framed环境（framed宏包）中
5.  整体放入figure环境中，成为浮动体

示例代码：

```latex
\begin{figure}[!htb]
        \lstset{language=C,
                numbers=left,
                numbersep=3pt,
                %frame=single,
                tabsize=2,
                xleftmargin=.05\textwidth,
                %xrightmargin=.05\textwidth,
                basicstyle=\footnotesize,
            }
\begin{framed}

\begin{minipage}[b]{0.45\columnwidth}
\begin{lstlisting}
int callee(int x){
    return x;
}
\end{lstlisting}
\end{minipage}
\rule{.1pt}{36mm}
\hspace{5pt}
\begin{minipage}[b]{0.3\columnwidth}
\begin{lstlisting}
int caller() {
    return callee(5);
}
\end{lstlisting}
\end{minipage}

\end{framed}
\caption{An example program}
\end{figure}
```

