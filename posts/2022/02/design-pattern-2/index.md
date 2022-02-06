# 读书笔记-设计模式-创建型模式


## 简单工厂模式 {#简单工厂模式}


### 场景 {#场景}

创建一系列类似的产品，这些产品有差异，但属于同一类


### 方案 {#方案}

1.  创建产品的基类，为抽象产品
2.  一系列具体的产品继承自抽象产品
3.  创建一个工厂类，该工厂知道所有的具体产品，可以根据传入的参数构造相应的具体产品，返回抽象产品
4.  客户不需要知道具体产品类，只需要知道工厂类以及具体产品，就可以请求工厂为之构造具体产品


### UML表示 {#uml表示}

{{< figure src="/ox-hugo/2022-02-06_14-59-16_screenshot.png" >}}


### 其他 {#其他}

这里的工厂类也可以简化为一个普通的创建函数


## 工厂方法模式 {#工厂方法模式}


### 场景 {#场景}

也是要创建同一类但有差异的产品，每个产品的创建过程都比之前要复杂一些。
那么如果用简单工厂模式，工厂类的设计逻辑就比较复杂。


### 方案 {#方案}

为每一个产品都配一个对应的工厂，这个工厂就只负责这一种产品

1.  创建一个抽象产品类和一个抽象工厂类
2.  具体的产品类继承自抽象产品类，具体的工厂类继承自抽象工厂类
3.  具体的产品和具体的工厂存在着一一对应关系，一个工厂只负责一种产品的构造
4.  客户不需要知道具体的产品，但需要知道具体的工厂，给具体工厂发请求就可以得到具体的产品了


### UML表示 {#uml表示}

{{< figure src="/ox-hugo/2022-02-06_15-21-44_screenshot.png" >}}


### 其他 {#其他}

在C++中可以利用模板将产品和对应的工厂联系起来（将产品作为工厂模板的参数）

{{< figure src="/ox-hugo/2022-02-06_17-06-34_screenshot.png" >}}


## 抽象工厂模式 {#抽象工厂模式}


### 场景 {#场景}

产品不再是只有同一类了，产品也有多类，但是这些不同类的产品又需要同时对客户提供


### 方案 {#方案}

一个具体工厂不再只生产一种具体产品了，而是负责生产一组产品。

1.  对于每一类产品，都创建一个抽象产品类
2.  对于每一个抽象产品类，都创建一系列具体产品类
3.  创建一个抽象工厂类
4.  创建一系列具体工厂类，每个具体工厂都负责一组产品的构造
5.  客户不知道具体产品，只知道抽象产品类，也知道具体工厂类，给具体工厂发请求就可以得到那一组具体的产品了


### UML表示 {#uml表示}

{{< figure src="/ox-hugo/2022-02-06_15-32-45_screenshot.png" >}}


## 建造者模式 {#建造者模式}


### 场景 {#场景}

一个复杂对象拥有多个组成部分，如汽车，它包括车轮、方向盘、发送机等各种部件。


### 方案 {#方案}

一步一步构造整个复杂对象。

1.  创建一个抽象类builder，和若干个具体builder
2.  builder负责构造复杂对象的各个部分
3.  创建一个具体类director，内部保存了一个builder的引用，但不知道builder具体是哪个类
4.  director负责构造整个复杂对象，而构造的方法是给自己关联的builder发消息，让builder把各个组成部分构造出来
5.  客户只需要知道具体的builder，把这个builder交给director，然后请求director构造复杂对象即可


### UML表示 {#uml表示}

{{< figure src="/ox-hugo/2022-02-06_16-19-54_screenshot.png" >}}


## 原型模式 {#原型模式}


### 场景 {#场景}

原型模式多用于创建复杂的或者耗时的实例，因为这种情况下，复制一个已经存在的实例使程序运行更高效


### 方案 {#方案}

给对象添加一个clone方法，用于实现对自身的拷贝。
需要注意的是clone的具体实现，是深拷贝还是浅拷贝，要视具体情况而定。


### UML表示 {#uml表示}

{{< figure src="/ox-hugo/2022-02-06_16-28-42_screenshot.png" >}}


## 单例模式 {#单例模式}


### 场景 {#场景}

有些类只需要唯一实例


### 方案 {#方案}

一个类能返回对象一个引用(永远是同一个)和一个获得该实例的方法（必须是静态方法，通常使用getInstance这个名称）；
当我们调用这个方法时，如果类持有的引用不为空就返回这个引用，如果类保持的引用为空就创建该类的实例并将实例的引用赋予该类保持的引用；
同时我们还将该类的构造函数定义为私有方法，这样其他处的代码就无法通过调用该类的构造函数来实例化该类的对象，只有通过该类提供的静态方法来得到该类的唯一实例。

多线程场景下要小心设计。


### UML表示 {#uml表示}

{{< figure src="/ox-hugo/2022-02-06_16-34-11_screenshot.png" >}}
