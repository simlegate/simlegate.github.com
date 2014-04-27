---
layout: post
title: "Clojure源码分析之sequential?"
description: ""
category: ""
tags: [Clojure]
---
{% include JB/setup %}

无意中在github看见Clojure的一个验证框架 [validateur](https://github.com/michaelklishin/validateur),就随便看了一下它的源代码，发现比较简单，想着就来复习一下Clojure的知识。

validateur库中的代码片段：
    
    (defn as-vec
      [arg]
      (if (sequential? arg)
        (vec arg)
        (vec [arg])))

这段代码能理解其大致功能就是将传入的参数转换成vector，但是`sequential?`确从来没有见过，所以就查了一下API，也随便分析一下这个函数的实现。

Clojure实现如下：

    ;; Returns true if coll implements Sequential

    (defn sequential?
     "Returns true if coll implements Sequential"
      {:added "1.0"
       :static true}
      [coll] (instance? clojure.lang.Sequential coll)))

原来它只是判断了一下`coll`是否是`clojure.lang.Sequential`实例而已。

但是`clojure.lang.Sequential`的实现又是什么呢？

在github的clojure仓库中找到了[具体实现](https://github.com/clojure/clojure/blob/0b73494c3c855e54b1da591eeb687f24f608f346/src/jvm/clojure/lang/Sequential.java)：

    package clojure.lang;
    
    public interface Sequential {
    }


原来只是一个Java的接口啊。

我们在回到`validateur`的代码片段中，如果我们传给`as-vec`的参数是一个`vector`,通过`(println (class [2]))`可以看出vector是`clojure.lang.PersistentVector`的实例, `PersistentVector`的大致继承关系如下：

      public class PersistentVector extends APersistentVector ...

      public abstract class APersistentVector extends AFn implements IPersistentVector ..
      
      public interface IPersistentVector extends Associative, Sequential ...

`PersistentVector`继承`APersistentVector`抽象类，`APersistentVector`实现了`IPersistentVector`, `IPersistentVector`继承了`Sequential`接口

所以：

    (sequential? [2]) ;; true

还有一些数据结构，比如`list`, 他们也实现或者继承了`sequential?`,所以：

    
    (sequential? (list 2)) ;; true

