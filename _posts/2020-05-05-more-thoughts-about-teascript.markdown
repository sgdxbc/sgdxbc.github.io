---
title: 关于TeaScript的更多想法
date: 2020-05-05 15:26
categories: thoughts
tags: TeaScript JavaScript 脑内编程
---

重装了电脑然后惊讶地发现，STZhongSong居然不是Windows自带的字体，于是往GitHub上传了一份。虽然点开网页就要先加载11MB的字体文件相当不友好，但是反正也没打算把这博客给谁看不是。

今天读了[用Rust写linux内核模块][1]的老哥写的[后传][2]，又受到了一次鼓舞。如果一门语言能像Rust一样让另一门语言（C）的原住民动力强大又无怨无悔地迁移过来，那么它必然也应该像Rust一样拥有成功的设计。面对JavaScript，TeaScript的特性基本都是以「摒弃迷惑行为」作为设计出发点的，我想在这个大方向下能做的如何应该就完全看我自己的水平了。

首先是关于内部状态的讨论。上一篇文章中说要用闭包来作为内部状态的定义方式，思来想去还是觉得太隐晦了。假如说一个对象收听十几种消息，或者说在追踪别人的代码追踪进一个方法里面去的时候，用户对于方法里出现的名字是怎么个来头是完全没有头绪的，这时候要用户去`spawn`上面的某一层作用域里面翻半天，搞懂这个名字是个什么来头的内部状态，我觉得这太不人道了。

如果在`spawn`当中专门设计一个定义内部状态的语法的话，那么也就意味着`spawn`应当禁止闭包功能。这样做有长远的好处，因为除非像C++那样要求用户把闭包的范围写出来，否则闭包对象的依赖关系可能很没有效率，另外还有异步等场合处理起来也麻烦的问题。考虑到我还没有设计匿名函数字面量，要不要就这么彻底把闭包特性删除呢？作为JavaScript的抗争者感觉这样有点大逆不道，但是JS的闭包本来也有点烦——就是那个循环生成闭包的例子（虽然用`const`解决了一下）。我想如果用对象表示闭包，然后在辅以可口的语法糖，这应该不算很过分吧。

将内部状态的语法明确地独立出来还有一个好处，那就是进一步强调了TeaScript对「对象」这个概念的理解。虽然之前写过了不过还是再复述一遍，对象是：
* 拥有内部状态
* 能够收听和分发消息

的实体。它具有两个重要的特性：
* 状态的内部性，只有对象自己收听消息的实现才能访问状态
* 消息分发的多态性，自己不能处理的消息交给自己的监督者

其中第一点和绝大多数语言都不一样。在流行的动态语言中，实例变量（内部状态）是属于实例对象的，而实例方法（收听消息）是属于类对象的，在类对象的定义过程中程序员的精神是分裂的，一进入方法体`self`就是实例对象的引用，一出了方法体就又在操作类对象了。这样紧凑的写法有利有弊，但是我想试试另一种思路。

在TeaScript中我没有过分强调「继承」，哪怕是原型链继承。因为我明确地把内部状态这个概念给「摘」了出来，它是完全不参与多态派发的，所以一个对象和它的监督者之间的关系非常不亲密，对象在创建的时候不需要从监督者那里复制任何属性（JS），不需要调用监督者的任何钩子（Python），更不用提包含一个完整的监督者（C++）了。对象与监督者之间也不能共享任何内部状态，一切的联系只能通过消息分发的方式，监督者可以给对象发消息，对象也可以给监督者发消息。这种对对象系统的严格限制是我写了很多Python以后的心得——虽然Python没有这么变态的要求但是我还是会遵循这些要求。之前我说过我想要把Rust的那套trait和struct系统学过来，但是最终发现如果没有一套静态系统支撑这个思路很难行得通，我觉得内部状态的内部性应该可以殊途同归地实现我的目的。

----

作为JavaScript的反抗者，首先要搞清楚JS/TS哪里不行。其实在这个问题上我就不太行——我就没写过正经八百的TS代码，所以一切都是我胡思乱想出来的。
* 基于类的抽象模式，这个大家都不行，只有我行
* 相等判断，空值还有内建的字典类型这些基础建设，这些从Python那里学过来就可以了
* 其它的历史遗留问题/品味问题，最起码一套整齐的API还是可以有的

这里专门提一下字面量的问题。我很长一段时间里都不喜欢字面量（从而导致我没法设计出任何语言，节省了大量用来造编译器的时间），原因是「不公平」：为什么一个浮点数就可以直接写下来，但是一个IP地址就只能调用函数来创造呢？但是这回我看开了，TeaScript至少会有
* 整数/浮点数
* 字符串
* 列表（常数级别的堆栈操作）
* 字典

这些字面量。运行时会提供这些字面量的实现对象。要不要允许用户自定义这些对象呢？我再想想。

反正一句话，能帮助我在浏览器里写出东西的特性我全都要。

[1]: https://zhuanlan.zhihu.com/p/137077998
[2]: https://zhuanlan.zhihu.com/p/137907908



