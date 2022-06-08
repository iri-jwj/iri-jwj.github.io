---
title: "Activity.setContentView后发生的事情"
date: 2022-06-08T20:46:05+08:00
lastmod: 2022-06-08T20:46:05+08:00
keywords: []
description: ""
tags: [Android]
categories: []
author: "iri.jwj"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: true
autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---

## activity.setContentView() 后发生的事情

有时候看到好些文章的标题都在说 activity 的<kbd>setContentView()</kbd>方法后会发生什么，今天（很早之前写下的）也刚好了解了一下全局替换样式的方法，做一个自己的总结；
下面的总结都是以 AppcompatActivity 来解释的

1. 先来看看 <kbd>setContentView()</kbd> 之前发生的事情：

   在我们调用 <kbd>setContentView()</kbd>之前我们都必须调用 super.onCreate() 方法, 在这个方法里需要注意的是最开头的几句：

   ```
   final AppCompatDelegate delegate = getDelegate();
   delegate.installViewFactory();
   delegate.onCreate(savedInstanceState);
   ```

   <kbd>delegate</kbd> 就是一个代理，它是一个抽象类，所有的实现都在<kbd>AppCompatDelegateImpl</kbd>里，在这里我们需要注意的是第二句 <kbd>installViewFactory()</kbd>

   里面的实现是，如果根据当前 context 获取的 layoutInflate 中的 factory 为空，则为 layoutInflate 设置 factory 为自己（ delegate 实现了 factory2 的接口）但是当 factory 不为空时，就不会再为 layoutInflate 设置 factory ，使得 factory 不会被替换，也就为后面的全局替换样式等需求做好了铺垫。

   代码如下

   ```
   @Override
   public void installViewFactory() {
       LayoutInflater layoutInflater = LayoutInflater.from(mContext);
       if (layoutInflater.getFactory() == null) {
           LayoutInflaterCompat.setFactory2(layoutInflater, this);
       } else {
           if (!(layoutInflater.getFactory2() instanceof AppCompatDelegateImpl)) {
               Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                       + " so we can not install AppCompat's");
           }
       }
   }
   ```

2. setContentView() 开始：

   调用顺序是<kbd>setContentView() -> delegate.setContentView()</kbd>

   在 delegate 中调用了 <kbd>LayoutInfate.from(context).inflate(resId,parent)</kbd>方法；顺便提一下这的里 parent 是 decorView 下的 FramLayout ，所以如果用 Android Studio 查看 view 的层级时会在自己的布局上看到好多层东西- -

   下面分析 inflate 方法里干了什么。这里我们跳过 layoutInflate 中 inflate 调用重载的 inflate 方法，直接到终点。（中间调用了一个方法把 xml 的 id 转换为了 xmlResourceParser）

   在最终的 inflate 方法里，做的事情就是找到 xml 中的头 -> 解析 merge 标签（如果有）-> 调用 createViewFromTag 方法创建 xml 中的根布局->调用 rInflateChildren() 方法创建子 view -> 调用 rInflate() -> 检测 merge 与 include 标签，然后依旧是调用了 createViewFromTag 方法

   在<kbd>createViewFromTag()</kbd>中，检测了 factory2 （以及 factory ）是否为空，不为空就调用<kbd>factory2.onCreateView</kbd>来创建 view。 当我们没有自己创建 factory 时，factory2 就是 delegate ，所以 onCreateView 最终调用的是 delegate 中的 createView 方法。如果所有的 factory 都为空，那么就会调用 layoutInflate 自身的 createView 方法，这里使用了反射的机制获取 View 的类来进行初始化。

   进入 delegate 中的 createView 时，调用 AppCompatViewInflater 中的 createView 方法，里面会根据 name 的类型来创建 view。但里面居然是用 switch()case 的方法来写。 如果是自定义 view 时，在 switch 语句块的最末尾调用的方法返回的是null,然后再在后面判断调用 onCreateView 方法，依旧使用反射机制。

   当所有都创建完成时,就会调用上面的讲到的 <kbd>parent.addView()</kbd>方法,将创建好的 view 加入.

3. 另外的想法

   在看到的文章中，通过在调用<kbd>super.OnCreate()</kbd>前创建一个 factory2 ，并实现自己的逻辑，来对创建的行为进行干预。这样后面 setContentView 就不会使用默认的 factory2，而是自定义的factory2。 这就需要看情况分类, 如果只需要对部分的样式修改， 那在自定义的factory2中还是要回到原本的创建流程，否则最终需要做的就是自己完成所有view的创建工作。使用场景上，现在已经有的包括:全局的主题修改、 代替 shape 和 seletor 、对特定的 view 作出改变等。
