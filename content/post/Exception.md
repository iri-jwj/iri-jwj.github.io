---
title: "Exception"
date: 2022-06-08T21:11:29+08:00
lastmod: 2022-06-08T21:11:29+08:00
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

# Exception

*这篇笔记是在有道上写好了的，现在归档到博客上*

Java中的Exception基本上是所有异常的父类，也就是基本上所有的异常都是直接、间接继承自Exception.

异常机制中包括两个步骤，一是异常的产生与抛出(throw)，二是异常的捕获与处理（try catch finally）

异常中主要的信息包括message和stacktrace，前者是异常本身的信息，后者是异常发生的调用栈信息，分析这两者可以去定位异常发生的位置以及原因等。同时，异常本身也是一个类，因此也可以附加一些其它属性以帮助我们更好地分析问题发生的原因以及发生时的状态。

使用的注意事项：
首先 异常抛出的定义在方法定义的后面，如 public void afunction() throws AException{}，一个方法后可以定义抛出多个异常。

基类方法和接口方法中抛出的异常：

首先，基类中的构造函数抛出的异常在子类的构造函数中必须也声明抛出，但构造函数的异常可以任意增加。

基类中abstract方法抛出的异常在子类的实现中可以不用显示声明，同时子类中继承的对应方法只可以增加子Exception（即基类中方法抛出的异常的子类）

从接口中实现的方法可以不用显示地声明抛出的异常，同时子类中继承的对应方法只可以增加子Exception（即基类中方法抛出的异常的子类）

```
public class StormyInning extends Inning implements Storm {

StormyInning() throws RainedOut, BaseBallException { //这里是构造函数，显示声明BaseBallException，同时也可以增加其它的Exception如RainedOut
super();
}

@Override
void atBat() {//从基类中继承的函数，可以不显示声明抛出的异常
//throw new Strike();
}

@Override
void Walk() {//从基类中继承的函数，由于没有在基类中没有声明抛出任何异常，因此也不能新增抛出的异常
super.Walk();
}

@Override
public void event() { }//这个方法在基类中和接口中都有定义，但继承的将是基类的方法，也就不会抛出任何异常

@Override
public void rainHard() throws ChildRainedOut, RainedOut { }//从接口中实现的方法，可以增加新的异常抛出声明，但新的异常必须是
//接口中定义的异常的子类
public static void main(String[] args) {
try {
StormyInning si = new StormyInning();
si.atBat();
si.Walk();
} catch (PopFoul p) {}
catch (BaseBallException e) { }
catch (RainedOut f) { }

try {
Inning i = new StormyInning();
i.atBat();
i.Walk();
} catch (Strike s) { }
catch (Foul f) { }
catch (RainedOut r) { }
catch (BaseBallException b) { }

try{
StormyInning si = new StormyInning();
si.event();
}catch (RainedOut r){}
catch (BaseBallException b){}
}
}

class BaseBallException extends Exception { }

class Foul extends BaseBallException { }

class Strike extends BaseBallException { }

abstract class Inning {
Inning() throws BaseBallException { }
abstract void atBat() throws Strike, Foul;
void event() throws BaseBallException { }
void Walk() {}
}

class StormException extends Exception { }

class RainedOut extends StormException { }

class PopFoul extends Foul { }

class ChildRainedOut extends RainedOut { }

interface Storm {
void event() throws RainedOut;

void rainHard() throws RainedOut;
}
```
