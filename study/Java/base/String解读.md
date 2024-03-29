# Table of Contents

* [String解读](#string解读)
    * [前言](#前言)
    * [String存储的是啥](#string存储的是啥)
    * [String的创建](#string的创建)
    * [String的存储](#string的存储)


# String解读

## 前言

String是Java中最常用的类之一，代表的含义就是字符串，正常用大家感觉都是唯手熟尔，也没啥特别的。但在面试中经常会出现一些问法让人瞬间转向，本篇就尽量解读以下String，让某些问题从根上不再让人混淆。


## String存储的是啥

任何数据最终存储都会归结到基本类型，String显然不是一个基本类型，其实它之中维护了一个char数组，这就是字符串的本体了。


## String的创建

以下罗列了两种比较常规（常规不是常见）的创建方式（不常规的就不考虑了，本篇不是百科全书）：

```
// 常见的方式
String s1 = "abc";
// 不常见但好像更好理解的方式 面试更是频繁露脸
String s2 = new String("abc");
```

- 第一种方式：基于有认知的人，把字符串abc赋值给String对象s1，没毛病，初学Java刚知道个对象的人一看，这是啥（黑人问号？？？）
- 第二种方式：大家一看都像那么回事，创建一个对象嘛，new看着一看就正规，结果一看源码又傻眼了（如下），这传入了一个String对象构造了一个String对象，没转过弯的当场去世，转过弯的一反应这不就又回到第一种方式了。

```
    // 源码
    public String(String original) {
        this.value = original.value;
        this.hash = original.hash;
    }
```

反正总之来说，一般形式最后的创建就归于如下方式（更严谨的说法这叫字面量赋值）：

```
String s = "xxx";
```

但其实还有种方式，下面分析会提及。创建是这么回事，用起来简单，那么是什么原理，创建方式不寻常，那字符串对象是否也像普通对象一样交由堆管理？


## String的存储

**（接下来是一堆不全是概念混合了理解的概念，并且基于java8）**

首先来讲讲字符串常量池（String Constant Pool），别和运行时常量池搞混，这是两个东西，字符串常量池可以理解为String的家，在Java8中其位于堆中（感兴趣的可以了解这里面的历史变动）。

基于这个认知，似乎可以得到一个结论，String是存储在堆内存的字符串常量池中，其实这么说不准确（而且是多方面的不准确）。

1. 只能说字面量赋值的String是字符串常量池在维护（并不是在其中）
2. 通过new的方式
    1. `new String(String)`方式首先会进行字面量赋值再生成一个不受限于字符串常量池的String对象
    2.  `new String(char[])`方式则会生成一个完全和字符串常量池无关的String对象

解释上面几点，那首先得知道字符串常量池究竟是啥，我们没有必要去深究这个东西，其实只是为了辅助我们理解。这里我就直接说了，字符串常量池的实现其实就是一个哈希表，你可以就理解为Java里`HashMap`的构造，在源码里就是类似`HashTable<const char*,obj>`这样的结构（需要稍微了解下c++的知识，看不懂点去搜索）。


对于字面量赋值方式，会初始化一个java_string对象并交于这个池子维护，简单理解就是一个key为字符串字面量，value为String对象（其实是引用，这不需要再讲了吧）的结构，所以其实对象本身还是在堆中，字符串常量池只维护其引用（插一句话，其实java8中所有的对象都在堆中，包括运行时常量池的常量也只维护引用，这一点是一致的）。

`new String(char[])`方式生成的对象为啥不在字符串常量池维护，其实也好理解，我直接生成一个对象，其内部char[]变量由我主动赋予，这个过程根本就没有涉及到字面量，也就不归字符串常量池维护，当然这里可以通过调用`String#intern`将值交于字符串常量池管理。

`String#intern`不多讲，它的作用就是取出字符串常量池中该字符串对象字面量的引用并返回，若是没有找到就把当前对象作为该字面量在字符串常量池中的对应引用。


所以总结就是，其实所有的String对象都是堆直接管理，参与正常的GC，但只要涉及到字面量（或者主动调用`String#intern`），String对象还会去字符串常量池维护一份引用（更像是个索引）。
