# anki基础-基本概念


要高效利用anki这个神器学习，需要知道几个基本概念：

1.  card
2.  note
    1.  field
    2.  card type
    3.  note type
3.  deck
4.  collection


## card {#card}

对应一张分为正面Q和反面A的卡片。


## note {#note}

对应一则信息。anki要做的，正是要把一则信息，转化为一张或多张卡片，方便用户记忆和学习。对应于这个转化过程的需要，故引入以下几个延伸的概念。


### field {#field}

将一条note结构化，分成几个field。比如，最简单的分法是分为两部分：问题Q和答案A。这个结构化的过程很重要，使零散的信息变得有条理，变得井然有序，不仅能加深对信息的理解，拆解信息本身也能促进记忆。


### card type {#card-type}

一条note的各个field最终要展示到card上，展示的方式由card type控制。card type实际上是一个模板，需要field的地方用对应的占位符填充。


### note type {#note-type}

一条note要拆解成哪几个field，展示方式（card type）有哪几种，是由note type控制的。如果card type有多个，那么一条note最终转化出的card会有多个。一条note必须有对应的一个note type。


## deck {#deck}

简单的deck就是一系列card的集合，相当于把一堆卡片放在一起，方便做记忆计划。当然也有复杂点的deck，就是一个deck包含了另一个subdeck，构成父子关系，形成嵌套结构。


## collection {#collection}

以上所有东西，集合到一起就构成了collection。可以将一个collection整体打包保存和发送，这是为了方便用户备份和迁移。

厘清了以上这些基础概念，也就理解了anki这个软件的设计逻辑。而只有了解了anki整个逻辑，才能更高效地使用它，在学习上事半功倍。


---

> 作者: [lazypanda](https://github.com/wanghuibin0)  
> URL: https://simplecoding.fun/posts/2021/10/anki-basics/  

