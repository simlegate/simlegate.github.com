---
layout: post
title: "[译文]Clojure之惰性"
description: "Clojure之惰性"
comments: true
category: ""
tags: [Clojure, Laziness]
---
{% include JB/setup %}

原文地址: [Being Lazy in Clojure](http://noobtuts.com/clojure/being-lazy-in-clojure)

## 前言

这篇文章主要阐述了Clojure的惰性以及给出了一个帮助我们解决很多计算问题的真实例子。

## 什么是惰性

简单地说就是惰性能够定义一些东西，并且在需要的时候才使用它们。

例子:

    (def nums (map inc [1 2 3]))

上面这段代码的作用是将序列`[1 2 3]`中的每一个元素加1，所以nums就为`[2 3 4]`，是这样吗？

事实上，这段代码没有进行任何的计算，知道我们使用`nums`，加1的计算才会被执行

    nums ;; => [2 3 4]

其实，大家都上面的原理。

下面让我们自己实现`inc`函数，它会做两个工作，打印出一个句号并且让整数`n`加1。这样可以让我们精确的知道计算操作被执行的时间。

    (defn pinc [n]
      (prn ".")
      (inc n))

我们再次定义`nums`

    (def nums (map pinc [1 2 3]))

在`REPL`中运行上面的代码，发现什么也没有被输出。说明`pinc`根本就没有被执行。

如果直接调用`nums`:

    nums ; => prints "..." and gives (2 3 4)

`...`被打印并且计算操作也被执行了。

这就是惰性计算。

## Clojure到底有多懒？
许多函数式编程语言都具有惰性。比如：`Haskell`语言就是完全的惰性。Clojure并非这样，只有大部分的序列操作才是惰性的，比如：**map**,**reduce**,**filter**或者**repeatedly**.

比如，下面的代码将直接被求值，因为它仅仅是定义了一个数字，没有包括任何序列：

    (def n (pinc 0)) ; => 直接打印出 "."
