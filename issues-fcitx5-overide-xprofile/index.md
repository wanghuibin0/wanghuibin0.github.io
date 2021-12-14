# 如何解决fcitx5覆盖.xprofile中按键设置的问题


## 问题的现象 {#问题的现象}

本来fcitx5正常。为了将caps lock映射为ctrl，我在.xprofile中放了这么几句：

```sh
setxkbmap -option ctrl:nocaps
setxkbmap -option shift:both_capslock
```

结果出问题了：本来，fcitx5在中文输入状态按shift可以字母直接上屏并切换为英文输入状态，现在不仅不能正常切换，而且配置的caps lock转ctrl也未生效。


## 问题的原因 {#问题的原因}

在[fcitx-FAQ](https://fcitx-im.org/wiki/FAQ)中看到这么一条：

> Fcitx now control keyboard layout and when switch layout, xmodmap setting will be overwritten.

<!--quoteend-->

> Since 4.2.7, Fcitx will try to load ~/.Xmodmap if it exists.

虽说这是针对fcitx说的，我觉得对于fcitx5应该是同样的道理。简单说，就是.xprofile中的设置被后续启动的fcitx5给覆盖了。


## 解决方案 {#解决方案}

在~/.Xmodmap中进行caps lock转ctrl的设置，因为fcitx5在启动时会兼顾到~/.Xmodmap,这样就不会无脑覆盖了。

```nil
remove Lock = Caps_Lock
keysym Caps_Lock = Control_L
add Control = Control_L
```

重启机器，一切正常了。

