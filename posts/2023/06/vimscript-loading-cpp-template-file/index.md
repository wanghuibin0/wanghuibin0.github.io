# Vimscript-一键生成脚手架代码


最近复习C++，会写一些简短的程序做实验，每次都要手动添加头文件，手动定义main，太麻烦，于是写了一点vimscript，可以一键自动生成这种重复性的脚手架代码。如下：

```vim
nnoremap <C-x>t :call <SID>ReadSourceTemplate()<CR>

function! s:ReadSourceTemplate()
  if &filetype == 'cpp'
    let l:template_file = '~/.config/vim/cpp_template.cpp'
  elseif &filetype == 'c'
    let l:template_file = '~/.config/vim/c_template.c'
  endif
  exe ':0 read ' . l:template_file
  exe '/main'
  exe 'norm! o'
endfunction
```

备注： 1.
真正的脚手架代码事先写好放在特定的模板文件里：'~/.config/vim/cpp_template.cpp'，
'~/.config/vim/c_template.c' 2.
nnoremap行定义了普通模式下的快捷键&lt;C-x&gt;t，用于触发一键生成脚手架文件的功能 3.
函数体内先判断当前文件名后缀，据此决定载入哪个模板文件 4.
接下来的三个execute将模板文件读入，然后自动定位到main的下一行

