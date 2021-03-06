---
layout: post
title: "所有权宣言 - Swift 官方文章 Ownership Manifesto 译文评注版"
date: 2017-02-27 15:40:00.000000000 +09:00
tags: 能工巧匠集
---

Swift 团队最近在邮件列表里向社区发了一封邮件，讲述了关于内存所有权方面的一些未来的改变方向。作为上层 API 的使用者来说，我们可能并不需要了解背后所有的事实，但是 Apple 的这封邮件中对 Swift 的值和对象的内存管理进行了很全面的表述，一步步说明了前因后果。如果你想深入学习和了解 Swift 的话，这篇文章是非常棒的参考资料。我尝试翻译了一下全文，并且加上了一些自己的注解。虽然这篇文章比较长，但是如果你想要进阶 Swift 的话，不妨花时间通读全文 (甚至通读全文若干遍)。

如果你没有时间通读全文，又想简单了解一下到底发生了什么的话，可以往下翻到最后，有一个我自己的简易的总结版本。

这篇文档本身是对今后 Swift 方向的一个提案，所以涉及的关键字和具体实现细节可能会有出入，不过这并不影响文章背后的思想。您可以在 Swift 的 repo 里找到[这篇文档的原文](https://github.com/apple/swift/blob/master/docs/OwnershipManifesto.md)。

这篇文章很长，读起来会比较花时间，不过因为内容还算循序渐进，只要静下心来就不会有太多困难。我在有些部分添加了个人的注解，会补充介绍一些背景知识和我自己的看法，你可以将它看成是我个人在读译本文时的笔记 (和吐槽)，它们将以 “译者注” 的方式在文中以引出出现。不过只是一家之言，仅供参考，还望斧正。

## 介绍

将“所有权”作为一个重要特性添加到 Swift 中，会为程序员带来很多好处。这份文档同时扮演了所有权这一特性的“宣言”和“元提案”的角色。在本文中我们会陈述所有权相关工作的基本目的，并描述要达到这些目的所使用的一般方法。我们还会为一系列特定的变化和特性进行提案，每个提案都会在将来在更细的粒度上分别进行讨论。这篇文档想要做的是在全局上提供一个框架，以帮助理解每个变化所带来的贡献。

### 问题现状

在 Swift 中广泛使用的值类型写时复制 (copy-on-write) 特性取得了很大成功。不过，这个特性也还有一些不足：

> 译者注：Swift 中的“写时复制”是指，值类型只在被改动前进行复制。传统意义上的值类型会在被传递或者被赋值给其他变量时就发生复制行为，但是这将会带来极大的，也是不必要的性能损耗。写时复制将在值被传递和赋值给变量时首先检查其引用计数，如果引用计数为 1 (唯一引用)，那么意味着并没有其他变量持有该值，对当前值的复制也就可以完全避免，以此在保持值类型不可变性的优良特性的同时，保证使用效率。Swift 中像是 `Array` 和 `Dictionary` 这样的类型都是值类型，但是底层实现确是引用类型，它们都利用了写时复制的技术来保证效率。

* 引用计数和引用唯一性的测试必然导致额外开销。

* 在大多数情况下引用计数可以决定性能特性，但是分析和预判写时复制的性能还是十分复杂。

* 在任何时候值都有可能被复制，这种复制会使值“逃逸”出原有作用范围，这会导致绝大部分底层缓冲区都会被申请在堆内存上。如果能在栈上申请内存的话，会比现在高效得多，但是这需要我们能够阻止，或者至少识别出那些将要逃逸的值。

有些低层级的程序对于性能有着更严格的要求。通常它们并不要求绝对的高性能，但是却需要**可以预测**的性能特性。比如说，处理音频对于一个现在处理器来说并不是什么繁杂的工作，就算使用很高的采样率一般也能应付自如。但是只要有一点点预期外的停顿，就会立刻引起用户的注意。

> 译者注：也就是说，相比于绝对的高性能，我们可能更希望有平稳的性能特性，来处理这些工作。避免代码性能上出现“尖刺”，让程序运行在可以预估的水平。这样一样，即使绝对性能不足，针对用户体验我们也可以很好的对策 (比如降低码率)，这可能比整体提升更重要，也更容易。

另一个很常见的编程任务是优化现有代码，比如你在处理某项工作时遇到了性能上的瓶颈。通常我们会找到执行时间或者是内存使用上的“热点”，然后以某种方式修复它们。但是当这些热点是由于隐式的值复制导致的话，Swift 中现在几乎没有工具能对应这种情况。程序员可能会尝试退回到用非安全指针来处理，但是这种行为将让你丧失标准库中集合类型所带来的安全性和表达能力的优势。

> 译者注：如果你觉得退回到非安全指针太过的话，也许退回到使用 `NSArray` 或者 `NSDictionary` 也会是一种选择。但是要注意数组或是字典中的类型最好也是 `NSObject` 子类，这样的回退才有意义。由于 Swift 中的类型和 Foundation 中的类型也存在一些隐式的桥接转换，这方面的性能开销往往被忽视。但是这样的妥协方式也并不理想，你同样失去了 Swift 的类型安全和泛型特性，同时可能你还需要大幅修改已有的模型类型，往往也得不偿失。

我们认为，通过引入一些可选的特性，我们将能够纠正这些问题。我们将这一系列特性统合称为**所有权**。

### 什么是所有权？

**所有权**是指某段代码具有最终销毁一个值的责任。**所有权系统**则是管理和转移所有权的一整套规则及约定。

任何具有销毁概念的语言，也都有所有权的概念。在像是 C 和非 ARC 的 Objective-C 这样的语言中，所有权是由程序员自行进行管理的。在其他一些语言中，像是一部分 C++ 里，所有权由语言进行管理。即使在隐式内存管理的语言中，也存在有所有权概念的库，这是因为除了内存以外，还有其他的编程资源，而理解那些代码应该释放那些资源，是一件非常重要的事情。

> 译者注：除了内存以外，其他的资源可能包括比如音频单元控制权、端口等等。这些资源也需要申请和释放，它们在这方面的运行逻辑和内存有相似之处。

Swift 已经有一套所有权系统了，但是它往往“鲜为人知”：这套系统是语言的实现细节，程序员几乎没有办法对其施加影响。我们想要提案的内容可以总结为以下几点：

- 我们应该向所有权系统中添加一条核心规则 - 独占性原则 (Law of Exclusivity)。这条原则应该阻止以互相冲突的方式同时访问某个变量 (比如，将一个变量以 `inout` 的方式传递给两个不同的函数)。这应该是一个必须的非可选改变，但是我们相信，这个改变对绝大多数的程序都不会产生不利影响。

- 我们应该添加一些特性，给程序员一定的手段来控制类型系统。首先，是允许被“共享”的值能传递下去。这将是一个可选的变更，它将由一系列的标注和语言特性组成，而程序员可以简单地选择不使用它。

- 我们应该添加一个特性，来让程序员表达唯一的所有权。换句话说，就是表达某个类型不能被隐式地复制。这将是一个可选的特性，能为那些想在这个层级进行控制的富有经验的程序员提供可行方式。我们不打算让普通的 Swift 程序也能够与这样的类型一同工作。

以此三个改变作为支柱，我们将要把这门语言的所有权系统从实现细节提升到一个更加可见的层面。这三个改变虽然优先级稍有不同，但是它们确是不可分割的，我们稍后会详述原因。由于这邪恶原因，我们将三者进行捆绑内聚，并将它们统称作“所有权”特性。

### 更多细节

Swift 现在的所有权系统中的基本问题在于复制，有关所有权的这三个改变全都在尝试避免复制。

在程序里，一个值可能会被用在很多地方。我们的实现需要保证在这些被用到地方，值的复制是存在且可用的。只要这个值的类型是可以复制的，那么我们只要复制这个值就肯定可以满足使用的需求了。但是，对于绝大多数的使用场景来说，实际上它们自己并不需要对复制的值拥有所有权。确实存在需要这么做的情况：比如一个变量并不拥有它当前的值，它只能将值存储在别处，而存储的地方是被其他东西持有的。但是这种做法一般并没有什么实际用处。而像是从类的实例中读取一个值这样的简单操作，只要求实例本身可用，而不要求读取代码自身实际拥有那个值的所有权。有时候这种差别十分明显，但是有时候却又难以分辨。举例来说，编译器在一般情况下是无法知道某个函数将对它的参数做怎样的操作的。在是否传递值的所有权这件事上，编译器只能回退到默认规则。当默认规则不正确的时候，程序就会在运行时多出额外的复制操作。所以，我们能做的是在某些方面让程序能被写得更加明确，这能帮助编译器了解它们是否需要值的所有权。

我们想要支持不可复制的类型，而这个方法和我们的想法是吻合的。对于大多数资源的销毁，唯一性十分重要：内存只能被回收一次，文件只能被关闭一次，锁也只能被释放一次。很自然地，对于这类资源的引用的所有者，应该也是唯一的，它不应该能被复制。当然，我们可以人为地允许所有权被共享，譬如我们可以添加一个引用计数，只在计数变为 0 时销毁资源，但是这势必会对使用这些资源带来额外的开销。更糟的是，这种做法会引入并行 (concurrency) 和[可重入](https://zh.wikipedia.org/wiki/可重入) (re-entrancy) 的问题。如果所有权是唯一的，并且语言本身强制规定了对资源的某些操作只能发生在拥有这个资源的代码中的话，那么自然而然地，同一时间内就只能有一段代码执行这些操作。而一旦所有权可以被共享，这一特性就随之消失了。所以，在一门语言里添加对不可复制的类型的支持会十分有意思，因为它能让我们以优秀和高效的抽象表达形式来操作资源。不过，要支持这些类型的话，我们需要完整应对抽象的所有方面，比如正确地标注函数的参数，以指明其是否需要所有权转移。如果标注不正确的话，只是使用增加复制的方式，编译器也无法保证在幕后一切都正确运行。

> 译者注：所有权唯一将会使资源管理的问题极为简化，但是事实上这样也会让程序变得无用。通过巧妙的语言设计 (或者说增加编译器开发者和语言开发者的压力)，可以在保持唯一性的同时与程序其他部分“共享”，不过这么做也会带来很多的复杂度。本文后面就将展示这些复杂度以及对应的方式。

想要将这些问题里的任何一个解决好，我们都需要解决变量的非独占访问 (non-exclusive access) 的问题。Swift 现在是允许对一个同样的变量进行嵌套式访问的。比如说，你可以将同一个变量作为两个不同的 `inout` 参数进行传递，或者是在一个变量上调用某个方法，并且在这个方法所接受的回调参数中再去访问同一个变量。这类行为基本上是不被鼓励的，不过它们也没有被禁止。不仅如此，编译器和标准库在这种时候都必须“卑躬屈膝”，以保证如果发生问题的时候程序不要表现得过于离谱。举例来说，在发生原地的元素替换更改时，`Array` 必须要持有它自己的内存。若不这样做的话，试想要是在更改的时候我们以某种方式把原来的数组变量重新赋了值，那么这块内存就将被释放掉，而元素却还正在被更改。同样地，编译器一般也很难证明某个内存里的值在一个函数中不同的地方是否相同，因为它只能假设任何一个非透明的函数调用都有可能重写内存。这导致的结果是编译器只能像一个被迫害妄想症患者那样到处添加复制，保证冗余。更糟糕的是，非独占访问极大地限制了显式标注的实用性。比方说，一个 `shared` 参数只有在保证该参数在整个方法调用中都有效时，才有意义。但是，只有通过对一个变量的当前值进行复制并传递复制的值，才能可靠地保证该值可以在可重入的方式下进行更改。另外，非独占访问也让特定的重要的模式变得不可能实现，比如无法“盗取”当前值并创建新值，在执行的途中别的代码要是可以获取某个变量的话，是一件很糟糕的事情。要解决这个问题，唯一的方法是建立一个规则，来阻止多个上下文在同一时间访问同一变量。这就是我们的提案之一 - 独占性原则。

所有这三个目标都是紧密相连，并且互为加强的。独占性原则使得显式标注在默认情况下能确实优化代码，并且对不可复制的类型进行强制规范。显式标注在独占性原则的作用下，可以带来更多的优化机会，并让我们在函数中使用不可复制类型。不可复制类型能够验证即使对于可复制类型来说，标注也是最优选项，它们也为独占性原则能直接适用创造了更多的情境。


### 成功的标准

如上所述，开发核心团队希望能将所有权作为可选的加强引入 Swift。程序员在很大程度上应该可以忽略所有权的问题，也不必为之操心。如果这一点被证明无法满足的话，我们会拒绝关于所有权的提案，而不会将这个明显的负担强加到普通的程序中去。

> 译者注：这实在是一个好消息。

独占性原则会引入一些新的静态和动态的限制。我们相信这些限制只会影响很小的一部分代码，而且这部分代码我们应该已经在文档中写明了可能产生非确定的结果。当我们进行动态限制的时候，还会造成一些性能上的损失。我们希望它所带来的优化潜力能够至少“弥补”这个损失。我们也会为程序员提供工具，来在必要的时候跳过这些安全检查。在文档后面的部分，我们会讨论很多这方面的限制。

## 核心定义

### 值

任何关于所有权系统的讨论都会基于更低的抽象层级。我们将要讨论的是一些语言实现方面的话题。在这个上下文中，当我们提到“值”这个词时，我们所表述的是具有特定语义的，用户口中的值的实例。

举例来说，下面的 Swift 代码：

```
  var x = [1,2,3]
  var y = x
```

人们通常会说这里 `x` 和 `y` 具有相同的值。让我们把这种值称为**语义值**。但是在实现层面上，因为变量 `x` 和 `y` 是能被独立改变的，所以 `y` 的值必须是 `x` 的值的复制。我们把这个叫做**值的实例**。一个值实例可以保持不变，并在内存中到处移动，不过进行复制的话则一定会导致新的值实例。在这篇文档剩下的部分，当我们不加修饰地使用“值”这个词时，我们指的是值实例这种更低层级的表述。

复制和销毁一个值实例的意义，随着类型不同稍有区别：

* 有些类型只需要按照字节表示进行操作，而不需要额外工作，我们将这种类型叫做**平凡类型** (trivial)。比如，`Int` 和 `Float` 就是平凡类型，那些只包含平凡值的 `struct` 或者 `enum` 也是平凡类型。我们在本文中关于所有权的大部分表述都不适用于这种类型的值。不过独占性原则在这里依然适用。

* 对于引用类型，值实例是一个对某个对象的引用。复制这个值实例意味着创建一个新的引用，这将使引用计数增加。销毁这个值实例意味着销毁一个引用，这会使引用计数减少。不断减少引用计数，最后当然它会变成 0，并导致对象被销毁。但是需要特别注意的是，我们这里谈到的复制和销毁值，只是对引用计数的操作，而不是复制或者销毁对象本身。

* 对于写时复制的类型，值实例中包含了一个指向内存缓冲区的引用，它的工作方式和引用类型基本相同。我们要再次提醒，复制值并不意味着将缓冲区中的内容复制到一个新的缓冲区中。

对每种类型，使用的规则是相似的。

> 译者注：在 Swift 中，值类型和引用类型的区别是相当重要的。当前 Swift 的最大的使用场景是和 Cocoa 框架合作制作 app，而 Cocoa 包括 Foundation 仍然是一个引用类型占主导地位的框架。值类型在 Swift 里使用非常广泛，你几乎很难避免混用两种类型。从 Swift 3 开始，开发团队正在将 Foundation 框架逐步转换为值类型 (比如 `NSURL` 到 `URL`，`NSData` 到 `Data` 的转换)，但是在底层它们包含了一个指向原来的对象类型的桥接。这就使得上面的最后一种情况 (写时复制类型) 变得非常普遍。另外，在我们自己创建的 Swift `struct` 和 `enum` 中，也经常会有引用类型作为成员的情况存在。而在这种情况下，写时复制并不是直接具备的特性，它需要我们进行额外的实现，否则我们就只能将它看作是引用类型来使用，否则很可能出现问题。在处理包含引用类型的值时，务必多多斟酌，特别小心。

### 内存

一般来说，一个值可以以两种方式中的一种被持有：它可能是“临时”的，也就是说一个特定的执行上下文对这个值进行了计算，并将它当作操作数；或者它可以是“静止”的，被存放在内存某处。

对于临时值，它们的所有权规则十分直接，我们无需多加关注。临时值是由一些表达式所创建的结果，这些表达式被用在特定的地方，而这些值也就只需要在被用在这些地方。所以语言实现需要做的事情就很清楚了：只需要完成将它们直接送到需要的地方就可以了，而不必强制对它们进行复制。用户已经明白会发生的是什么，因此这部分没有实际需要改进的必要。

那么，我们关于所有权的讨论将很大程度上围绕内存中保存的值来进行。在 Swift 中，关于对内存的处理，有五个紧密相关的概念。

**存储声明** (storage declaration) 是一个语法概念，它声明了相关内存在这门语言里被处理的方式。现在，存储声明通过 `let`，`var` 和 `subscript` 引入。存储声明是带有类型的，它也包含了一些定义，来规定读取和写入存储时的方式。`var` 或者 `let` 的默认实现除了创建一个新变量来存储值以外并没有做什么。不过存储声明也可以是被计算出来的，也就是，并没有必要说一个变量背后一定会有对应的存储。

**存储引用表达式** (storage reference expression) 也是一个语法概念，它是一个对存储进行引用的表达式。这个概念与其他语言的 "l-value" 比较相似，不过不同的是它不需要一定被用在赋值语句的左边，因为存储也可能是不变的。

> 译者注：本文中会多次提到存储引用表达式，所以为了确保能理解什么是存储引用表达式，我在这里啰嗦几句。所谓的 l-value 表达式，指的是一个指向某个具体存储位置的表达式。在其他语言中，l-value 应该是可以被赋值的，而在 Swift 中，l-value 并不需要能被赋值，是因为有常量值的存在，存储将不会改变。举例来说，比如对某个点 `Ponit` 的坐标的访问表达式 `p.x` 就是一个存储引用表达式，它访问的是具体存储的 x 值。如果 `x` 是以 `var` 的形式声明的，那么它和其他语言的 "l-value" 就是等同的，如果 `x` 的定义方式是 `let`，则不可赋值，但这并不影响 `p.x` 作为存储引用表达式存在。另外，如果正方形有一个面积计算属性 `var area: Float { retrun side * side }`，`square.area` 则不是存储引用表达式，因为它求值后并不是对存储的引用。

**存储引用** (storage reference) 是一个语言语义的概念，它声明了一个指向特定存储的完全填满的引用。换句话说，它是存储引用表达式抽象求值后的结果，不过它并不会实际去访问存储。如果存储是一个成员，那么基本会包含值或是指向存储的引用。如果存储是一个下标 (subscript)，它将包含索引的值。比如，像是 `widgets[i].weight` 这样的存储引用表达式可能被抽象求值为下述存储引用：

* 属性 `var weight: Double` 的存储
* 下标 `subscript(index: Int)` 在索引值 `19: Int` 位置的存储
* 本地变量 `var widgets: [Widget]` 的存储

**变量** 是一个语义概念，它指的是内存中存储一个值的唯一地点。变量不需要是可变的 (至少在我们的文档中它不需要可变)。通常来说，存储声明是变量被创建的原因，不过它们也会在内存中被动态地创建 (比如使用 `UnsafeRawPointer`)。变量总是属于某一个特定的类型，也有一定的**生命周期**，在编程语言中，生命周期是指从变量开始存在的时间点到它被销毁的时间点之间的时间。

**内存地址** (memory location) 是指内存中一连串的可以被标记位置的范围。在 Swift 里，这里基本上是一个实现细节上的概念。Swift 不保证任意的变量会在它的生命周期中都保持在同一个内存地址上，Swift 甚至不能保证变量一定是被存储在内存地址上。但是也有将变量临时强制放置在一个不变的地址上的时候，比如以 `inout` 方式将变量传递给 `withUnsafeMutablePointer` 时就遵循这条规则。

### 访问

对于存储引用表达式的某种特定的求值被称为访问。访问的方式有三种：**读取**，**赋值**，以及**修改**。赋值和修改都是**写操作**，不同的是赋值会将原来的值完全替换掉，而不会去读取它。修改的话需要依赖旧值。

所有的存储引用表达式都可以基于表达式出现的上下文，被归类到这三种访问类型中的一种。需要注意，这种归类是表面上的工作：它只依赖于当前上下文的语义规则，而不会去在程序的更深层次进行考虑和分析，也不会关心动态行为。比如，通过 `inout` 参数传递的存储引用不会关心被调用者有没有实际使用当前值，也不关心到底有没有实施写操作或者只是简单地使用它，这个访问在调用者里总是会被判断为更改访问。

存储引用表达式的求值可以分为两个阶段：首先会求得一个存储引用，之后，对存储引用的访问会持续一段时间。这两个阶段通常是接连进行的，但是在复杂的情况下，它们也可以被分开单独执行，比如在 `inout` 参数不是调用的最后一个参数时，就会发生这种情况。阶段分离的目的是将访问的持续时间最小化，而同时保持适应 Swift 的从左到右的最容易进行扩展的求值规则。

> 译者注：如果能够接受之前的五个概念的分别的话，将表达式的求值和使用 (对存储的访问) 过程分开处理也就是自然而然的事情了。虽然这让心智模型变得更加复杂，但是却能对应更多的使用情况，而且相对而言付出的代价可以接受。

## 独占性原则

建立起这些概念后，我们就能简要地提出这提案的第一个部分 - 独占性原则了。所谓独占性原则，是指：

> 如果一个存储引用表达式的求值结果是一个由变量所实现的存储引用，那么对这个引用的访问的持续时间段，不应该与其他任何对这个相同变量的访问持续时间段产生重合，除非这两个访问都是读取访问。

这里有一个地方故意说得比较模糊：这条原则只指出了访问“不应该”产生重合，但是它没有指出如何强制做到这一点。这是因为我们将对不同类型的存储使用不同的方法来强制这个机制。我们将在下一个大节里讨论那些机制。首先，我们想要谈一谈这条规则会带来的一些结果，以及我们满足这条规则所要使用的策略。

### 独占性的持续时间

独占性原则说的是访问在它们的持续时间内必须是独占的。这个持续时间是由导致该次访问的直接上下文所决定的。也就是说，这是程序的一种**静态**特性，而从介绍部分中我们知道，横在我们面前的安全问题是一个**动态**的问题。按照一般经验，我们知道使用静态的方式来解决动态问题往往只能在保守的范围内生效；在动态的程序中，肯定会存在方案失效的时候。所以，一个自然的问题是，在这里要如何才能让一个通用的原则生效。

举例来说，当我们用 `inout` 参数的方式来传递存储的时候，访问会贯穿与整个调用所持续的过程中。这需要调用方保证在调用过程中没有其他对这个存储进行访问。这么一刀切的手法会不会有点太过粗糙？因为毕竟在被调用的函数中可能会有很多地方其实并不会用到这个 `inout` 参数。也许我们应该在追踪 `inout` 参数的访问这件事情上再细一些，我们可以在被调用的函数中来进行追踪，而不是粗暴地在整个调用者上施加独占性原则。其实问题在于，这个想法实在是太动态了，所以我们很难为它提供一个高效的实现。

调用方的 `inout` 规则有一个关键的优点；对于被传递的存储到底是什么，调用方有着大量的信息。这意味着调用方规则通常能以纯静态的方式让独占性原则适用，而不必添加动态检查或是做一些猜疑性质的假设。比如，要是有一个函数调用了某个本地变量上的 `mutating` 方法 (`mutating` 方法实际做的就是将 `self` 作为 `inout` 参数传入)，除非变量被一个逃逸闭包 (escaping closure) 所捕获，否则函数就能轻而易举地检查每次对变量的访问，并确认这些访问和调用没有重叠，以此来保证满足独占性原则。不仅如此，这个保证还能被向下传递给被调用者，被调用者可以使用这些信息来证明它自己的访问是安全的。

> 译者注：我们往往会认为实际工作中 `inout` 的使用非常罕见，这说明你忽视了 `mutating` 方法的实质就是 `inout` 参数调用。在标准库中，很多关于数组或者字典的变更的方法都是 `mutating` 的，也符合这个原则。

相反，被调用方对于 `inout` 的规则却无法从这样的信息中受益：这些信息在调用发生的时候就被抛弃了。这就造成了我们在介绍一节中谈到的现今普遍的优化问题。比如，假设被调用者从参数加载了一个值，然后调用一个优化器无法进行推断的函数：

```
  extension Array {
    mutating func organize(_ predicate: (Element) -> Bool) {
      let first = self[0]
      if !predicate(first) { return }
      ...
      // something here uses first
    }
  }
```

在被调用方的规则下，优化器必须把 `self[0]` 的值复制到 `first` 中，因为它只能假设 `predicate` 有可能以某种形式改变 `self` 上绑定的值。在调用方规则下，优化器则能在数组没有被改变的时候，一直使用数组中的元素值，而不需要进行复制。

不仅如此，试想上面的例子里，如果要遵守被调用方规则的话，我们能写的代码会变为怎样呢？像这样的高阶操作不应该需要担忧调用者传入的 `predicate` 会以再入的方式改变数组。上面例子里，像使用本地变量 `first` 而不是反复地访问 `self[0]` 这样简单的实现选择，在语义上就会变得特别重要；而想要维护这种事情的难度是不可想象的。所以 Swift 库一般都禁止这种再入式的访问。不过，因为标准库并不能完全地阻止程序员这么做，所以实现必须在运行时做一些额外的工作，来确保这样的代码不会导致未定义的行为，或者是让整个进程发生错误。如果作出限制的话，这些限制只会作用于那些在良好书写的代码中本不应该出现的情况，所以对大多数程序员来说，并没有什么损失。

因此，这个提案提出了类似调用方的 `inout` 那样的访问持续时间的规则，它能让接下来的调用有机会被优化，同时保证需要付出的语义代价很小。

> 译者注：也就是说，所需要修改的代码很少，代码“变丑”或者“变复杂”的程度在可控范围之内。

### 值和引用类型的构成

我们已经谈了很多关于**变量**的内容了。读者可能会想要知道，**属性** (property) 的情况是如何的。

在我们上面陈述的定义体系中，属性是一个存储声明，一个存储属性会在它的容器中创建一个对应的变量。对这个变量的访问显然需要遵守独占性原则，但是因为属性是被组织到一起放在一个容器中的，这会不会导致一些额外的限制？特别是独占性原则应不应该阻止那些对同一个变量或者值，但是是通过不同属性所进行的访问。

属性可以被分为三种类型：
- 值类型的实例属性，
- 引用类型的实例属性，以及
- 在任意类型上的 `static` 和 `class` 属性。

> 译者注：相比于 Objective-C，Swift 中的属性似乎并不是特别明显。因为 Objective-C 毕竟有 `@property` 这种显式的方式声明属性，而在 Swift 中，写在具体类型而非方法中的“变量声明”将自动成为属性。

我们提议总是将引用类型属性和 `static` 属性看作各自独立的属性，而在某个特定 (但是很重要) 的特殊情况以外，将值类型的属性当作是非独立的来进行处理。这可能会带来很大的限制，对为何这个提议是必要的，以及为什么我们对不同类型的属性加以区别，我们会详加说明。主要有三个原因。

#### 独立性和容器

第一个原因和容器有关。

对值类型来说，访问单个的属性和访问值的整体都是可能的。显然，访问一个单个属性和访问值的整体是冲突的，因为访问值的整体其实就是同时访问这个值里所有的属性。举例来说，比如有一个变量 `p: Point` (这个变量并不需要是一个本地变量)，它包含三个存储属性 `x`，`y` 和 `z`。要是能够同时并且独立地改变 `p` 和 `p.x` 的话，独占性原则就会有一个巨大的漏洞。所以我们必须在这里强制独占性原则，我们有三个选择：

(在阅读关于强制适用独占性原则的部分后，再来看这节内容会更容易理解。)

第一种方法是简单地将 `p.x` 的访问也看作是对 `p` 的访问。这很巧妙地就将漏洞堵上了，因为我们对 `p` 所适用的独占性原则很自然地将冲突的访问排除了。但是这也同时将对于其他属性的同时访问给排除了，因为对 `p` 中其他属性的访问都会触发对 `p` 的访问，从而使独占性原则生效。 

另外两种方法需要让这种关系倒过来。我们可以将所有对单独的存储属性的独占性原则分离出来，而不是对整体值进行独占：对于 `p` 的访问会被看作是对 `p.x`，`p.y` 和 `p.z` 的方式。或者我们可以将独占性适用的方式参数化，并且记录正在被访问的属性的路径，比如 ""，".x" 之类。不过，这些方式存在两个问题。

首先，我们并不总是知道全部的属性，或者那些属性是存储属性；某个类型的实现对我们来说有可能是不透明的，比如泛型或者还原的类型。对计算属性的访问必须被当作对整个值的访问，因为它需要将变量传递给 getter 或者 setter，而这些访问方法会是 `inout` 或者 `shared` 的。所以实际上它是和其他所有属性冲突的。使用动态的信息是可以让它们正常工作，但是这会在值类型的访问方法中引入很多记录方法，这和值类型被作为低耗费的抽象工具这一核心设计目标大相径庭。

其次，虽然这种模式可以相对容易地应用在独占性的静态适用上，但是想用在动态中就需要一大堆动态的记录，这也与我们的性能目标格格不入。

> 译者注：这里考虑的属性访问路径和 Objective-C 的 KVC 有形似之处。不过这也正是 Swift 所极力避免避免的问题。类似这样的标注，或者说动态的修改，对于性能的损失是不可忽视的，在尽可能的情况下，我们都希望适用静态的方法来保证独占性。只有在确实无法静态决定的情况下，再使用动态方式。

所以，虽然有方法能让我们将对不同的属性的访问和对整体值的访问独立开来，但是这要求我们强制对整体值使用独占性原则，而且还需要两种属性都是存储属性。虽然这是一种很重要的特殊情况，但是它也仅仅只是一种特殊情况。对于其他情况，我们必须回退到一般的规则，认为对于一个属性的访问同时也是对整体值的访问。

这些思考对于 `static` 属性以及引用类型的属性是不适用的。在 Swift 中没有同时访问一个类中所有属性的语言结构，而且说一个类型所有的 `static` 属性本身就是没有意义的事情，因为任何一个模块都能够在任何时候向某个类型添加 `static` 属性。

#### 独立访问的具体表现

第二个原因是基于用户期望的考虑。

避免对于不同属性的重叠访问最多只会造成一些小麻烦。在独占性原则下，我们至少可以避免“让人惊讶的长距离访问”：调用变量上的一个变量有可能开启一长串不明显的事件序列，然后最后返回并修改了原来的变量。现在有独占性原则，这就不会再发生了，因为这将会导致两个对同一变量互相冲突且重叠的访问。

作为对比，引用类型中有许多已经成为习惯的模式正是依赖这种“基于通知”的方式进行更新的。实际上，在 UI 代码中，同一个对象上的不同属性被并行修改并不是一件罕见的事儿：比如被用户 UI 操作修改的同时，被一些后台操作修改。阻止独立访问将会打破这种做法，这是无法接受的。

对于 `static` 属性，程序员期望它们是独立的全局变量；在一个全局变量被访问的时候，去禁止别的访问访问，这是说不通的。

#### 独立性和优化器

第三点和属性的优化潜力有关。

独占性原则的一部分目标是让一大类的优化能够适用于值。比如，值类型上的一个非 `mutating` 方法可以假设 `self` 在方法调用期间会完全保持一致。它不需要担心某个未知的函数会在调用期间返回并修改了 `self` 的值，因为这种修改将违背独占性原则。即使在 `mutating` 的方法中，除非知会这个方法，否则也没有其他代码能访问 `self`。这些假设对于优化 Swift 代码是非常关键的。

不过，这些假设一般来说对全局变量和引用类型属性的内容来说都不适用。类的引用可以被随意地共享，所以优化器必须假定某个未知方法可能会访问到同一个实例。另外，系统中的任何代码 (如果忽略访问权限控制的话) 都有可能访问到全局变量。所以就算将对不同属性的访问当作非独立的来对待，语言的实现能获得的好处也及其有限。

#### 下标

虽然现在这门语言中在技术上来说下标从来不会是存储属性，但大多数的讨论对下标依然适用。通过下标访问值类型的一个部分和访问整个值是同等对待的，对和这个值的其他访问发生重叠的时候，也应如此考虑。这导致的最重要的结果就是两个不同的数组元素不能被同时访问。这会妨碍到某些操作数组时的通常做法，不过有些 (比如并行地改变一个数组的不同切片这类) 事情在 Swift 中本来就充满了问题。我们认为，通过有目的的对集合的 API 进行改进，可以将缓和所带来的主要影响。

> 译者注：在日常开发中，对于数组的操作可能是并行编程中比较常见的多线程问题。在很大程度上，下标操作和实例属性的访问类似，我们可以通过加锁或者 GCD 做 barrier 的方式来确保数组线程安全。如果能在语言层面将独占性原则解决的话，将极大程度降低并行程序开发的难度。这也意味着今后的标准库中我们可以获得线程安全的数组，或者甚至整个标准库乃至第三方代码都会是默认线程安全的。

## 独占性原则的强制适用

想要让独占性原则适用，我们有三种可行机制：静态强制，动态强制，以及未定义强制。所要选择的机制必须能由存储声明简单地决定，因为存储的定义和对它的所有直接的访问方法都必须满足声明的要求。一般来说，我们通过存储被声明为的类型，它的容器 (如果存在的话)，以及任何存在的声明标记 (比如 `var` 或者 `inout` 之类) 来决定适用哪种机制。

### 静态强制

在静态强制的机制下，编译器将检查独占性原则是否被违反，如果违反，则给出编译错误。因为这种方法安全可靠，并且不会产生运行时的损耗，所以应该是优先考虑的机制。

这种机制只能在所有东西都能被完美决定时使用。比如，对于值类型，因为独占性原则递归适用于所有属性，这保证了基本的存储是被独占访问的，所以独占性原则可以适用。对一般的引用类型则不适用，因为无法证明对于某个特定对象的引用是对这个对象的唯一的引用。不过，如果我们能够支持唯一引用的 class 类型的话，独占性原则就可以静态地适用它们的属性了。

在一些想定的情况下，编译器可能会为了保持源码兼容性和避免发生错误，从而隐式地插入复制操作。这应该只在需要源码兼容的模式下被使用。

> Swift 4 的路线图已经发布，主要会在 `String` 的部分引入部分的源码非兼容改动。同时 Swift 4 的编译器会支持通过特定的编译标记来在文件粒度上支持选择按 Swift 3 还是 Swift 4 进行编译。这种方式比现在 Swift 2.3 和 Swift 3 共存的方式要进步一些，不过也并不是说 Swift 4 就不用做迁移了...不过看情况似乎会比 2 到 3 的时候容易很多，因为至少我们可以做到一个文件一个文件进行迁移。

静态强制会被用在：

- 各类的非可变变量

- 本地变量，除非被闭包的使用影响 (之后会详述)

- `inout` 参数

- 值类型的实例属性

### 动态强制

在动态强制下，语言实现将会维护一个记录，来确定每个变量现在是否正在被访问。如果发现冲突，它就将触发一个动态错误。如果编译器侦测到动态强制下一定会发现冲突的话，也可以由编译器给出一个静态错误。

进行记录时，对于每个变量需要两个 bit，将它标记为“未访问”，“读取”和“已修改”三种状态中的一种。虽然多个读取操作能够在同时一起生效，不过只要做法稍微聪明些，我们就可以通过在访问时将原来状态进行存储的方式，来避免对所有的访问进行完全的记录。

我们应该尽最大努力进行记录。这个做法需要能可靠地检测出那些必然会违反独占性原则的情况。我们没有要求它检测竞态条件 (race condition) 的状况，不过好消息是虽然没做要求，但它通常还是可以检测出竞态，这是一件好事。记录**必须**要能正确处理并行读取的情况，而不应该比如将记录永久地停在“读取”状态。但是，在并行读取时，即使是还有活动的读取者，将记录值设为“未读取”的状态却是可以接受的。这可以让记录使用非原子 (non-atomic) 操作。不过，在将一个类的不同属性的记录值打包到一个单独的字节时，却必须使用原子操作，因为在一个类中，对不同变量的并行访问是允许的。

> 译者注：对于对 Objective-C 不熟悉的读者来说，原子操作和非原子操作可能是比较陌生的概念。原子操作指的是不会被线程调度机制打断的调用。比如在满足原子操作的 getter 中，同一个属性的 setter 被调用，那么 getter 还是能返回完整的正确值，但是非原子操作的属性则不具备这个特性，因此非原子要快得多。Swift 现在没有设置原子属性的语法，所有的属性默认都是非原子操作。如果需要在 Swift 中让属性满足原子操作，现在我们可能需要自行进行加/解锁。另外注意，原子/非原子操作和线程安全的程序并没有太大关系，它只是针对一个属性的一次读写操作所做的特性设定。

当编译器检测到一个“实例内部”的访问时，也就是说在这种情况下，访问期间没有别的代码会执行，也就没有对同一个变量进行再入式访问的可能。此时，编译器就能避免更新记录的值，而只需要检查它现在是否具有一个恰当的值。这对读取来说很正常，因为读取操作往往只会在访问过程中复制值。当变量是 `private` 或者 `internal` 时，编译器可以检测到所有可能的访问都是内部访问，它就能够将所有的记录都去掉。我们希望这应该是非常常见的情形。

动态强制会被用在：

- 使用了闭包，且有必要时的本地变量 (之后会详述)

- class 类型的实例属性

- `static` 和 `class` 属性

- 全局变量

我们需要为动态强制提供一个标注，来让它在特定的属性和类中降级去使用未定义强制的机制。当有人觉得动态强制的性能消耗太重时，这可以为他们提供一种将这个特性去除的方式。在独占性实现后的早期阶段，留有余地是尤其重要的，因为我们可能还在探索不同的实现方法，有可能还没有找到全面的优化方式。

在之后，我们可以对类实例进行隔离，这让我们可以对一些类实例的属性使用静态强制。

### 未定义强制

未定义强制的意思是冲突既不会被静态检测，也不会被动态检测，冲突的结果将是未定义行为。对于 Swift “默认安全”的设计下的一般代码来说，这不是一个好的机制，但是这确实是像不安全指针 (unsafe pointer) 这类东西的唯一真正选择。

> 译者注：`Unsafe` 家族继续在 Swift 中扮演垃圾桶角色。对于 C 的库，如果没有更好的替代，可能我们也只能接受牺牲安全性的事实。但是还是建议在真的要在 Swift 中使用 C 库之前，再三斟酌。诚然花力气去把 C 库用 Swift 进行重写不是一件很讨好的事情，但是还是应该在代码的安全特性和直接使用 C 库的便捷程度中进行权衡。如果选择使用 C 代码，建议尽量使用 struct 来进行一定的封装，避免过多地使用 Unsafe 类型来进行交互。

未定义强制将被用于：

- 不安全指针的 `memory` 属性。

> 译者注：Swift 1 和 2 里是 `memory` 属性，Swift 3 中应该已经被改名为 `pointee` 了。如果说是 Swift 团队又打算把它改回 `memory` 的话...我也很无语。希望只是原作者的笔误。

### 被闭包捕获的本地变量的独占性

独占性原则的静态强制依赖于我们能够静态地知道对变量的使用发生在哪里。对于本地变量来说，这种分析通常都很直接，但是当一个变量被闭包所捕获后，控制流就会让使用情况变得难以理解，从而使整个事情变复杂。就算是非逃逸的闭包，也是有可能被重入或者并行执行的。对于闭包捕获的变量，我们采取以下原则：

- 如果闭包 `C` 有可能逃逸，那么假设有被 `C` 捕获的任意变量 `V`，对 `V` 的有可能在一段逃逸时间后才被执行的访问 (也包括在 `C` 本身中的访问)，都必须遵守动态强制的原则，除非所有的访问都是读取访问。

- 如果闭包 `C` 不会逃逸出函数，那么它在函数内的使用地点都是已知的。在每处使用时，这个闭包要么被直接调用，要么被作为参数传递给另一个调用。对于每次这种调用时的非逃逸闭包，对每个被闭包 `C` 所捕获的变量 `V`，如果任意一个闭包含有对 `V` 的写操作，那这些闭包内的所有的访问都必须使用动态强制，并且这个闭包调用会被静态强制认为是试图对 `V` 进行写操作的调用。除此之外，所有的访问都可以使用静态强制，而且对闭包的调用会被视作对 `V` 的读取操作。

可能这些规则会随着时间而逐渐改进。比如我们应该可以对闭包的直接调用的规则进行一些改善。

## 所有权使用的具体的工具

### 共享值

本章中的很多讨论里会出现一个新概念：**共享值** (shared value)。正如其名，共享值指的是一个被当前上下文和拥有它的另一部分程序所共享的值。因为程序的多个部分能够同时使用这个值，所以为了贯彻独占性原则，这个值对于所有上下文 (包括拥有这个值的上下文) 来说，都必须是只读的。这一概念可以使程序对值进行抽象，而不必对它们进行复制。这和 `inout` 可以让程序对变量进行抽象有异曲同工之妙。

> 译者注：程序对值或者变量进行抽象可能不太容易理解。可以参考 `inout` 的实现方式，其实 `inout` 使程序在函数返回前对传入的参数进行赋值操作，就是一种抽象行为：这个关键字 `inout`，将传入变量且在返回时重新赋值变量这个操作，抽象为了一个修饰词。下述的 `shared` 与此类似，只不过它所抽象的目标对象是值。

(熟悉 Rust 的读者可能会在共享值和 Rust 的不变借入 (immutable borrow) 的概念之间找到相似之处。)

> 译者注：不愧是把大半个 Rust 团队挖过来了...本来还打算今年学一下 Rust，现在看来把 Swift 4 学好就行了...最近三年果然还是坚持了每年学一门新语言，它们分别叫 Swift 2，Swift 3 和 Swift 4。

当一个共享值的源值是存储引用时，共享值实际上就是一个对存储的不可变引用。存储在共享值持续期间被作为读取操作进行访问，这样依赖，独占性原则会保证在访问期间不会有其他访问能对原来的变量进行修改。有些类型的共享值还可能被绑定到临时值上 (比如一个 r-value)。因为临时值总是被当前执行上下文所拥有，而且只在一个地方被使用，所以这不会带来额外的语义上的问题。

就像是普通的变量或者是 `let` 绑定时那样，我们可以在进行绑定后的作用域中使用共享值。如果要使用共享值的地方也要求所有权的话，Swift 将简单地对这个值进行隐式的赋值 - 这和普通的变量或者 `let` 绑定依然是一样的。

#### 共享值的局限

文档的这个部分将描述几种生成和使用共享值的方式。不过，我们现在的设计还没有提供一个通用的，“一等公民”的机制来使用共享值。程序并不能返回一个共享值，不能构建一个共享值组成的数组，也不能将共享值存储为 `struct` 的字段等。这些限制和 `inout` 引用所存在的限制是相似的。事实上，它们之间的相似非常多，因此我们甚至可以引入一个术语来包含它们两者：我们将它们叫做**暂态量** (ephemerals)。

我们的设计没有给暂态量提供完备的支持，这是精心考量的决定，这主要是基于三点考虑：

- 我们需要将这个提案的范围限制在未来几个月内可以确实实现的范围内。我们希望这个提案能给语言及其实现带来大量好处，但是提案本身已经涉及广阔，而且略有激进了。对暂态量的完整支持将会给实现和设计增加很多复杂度，这显然会导致提案超出预计范围。另外，余下的语言设计问题都很庞大，而且已经有几个现存的语言尝试了将暂态量作为一等特性，不过它们的结果并不能说完全令人满意。

- 类型系统是在复杂度与表述清晰度之间的权衡。只要将类型系统做得更复杂，你就总是能接受更多的程序，但是这并不一定是好的权衡。在像 Rust 这样的重视引用的语言中，背后的生命周期限定 (lifetime-qualification) 系统向用户模型中添加了很多复杂度。这些复杂度对用户来说确实成为了负担。而且它依然不可避免地时不时要回退到不安全的代码，来绕开所有权系统的一些限制。在当前看来，将作用域限定引入 Swift 并不是一个可以直接做出的决定。
  
- 在 Swift 中，一个类似 Rust 的生命周期 (作用域) 系统的功能并不需要像 Rust 中那样强大。Swift 有意地提供了一个让类型的作者和 Swift 编译器本身都能够保留很多实现自由度的语言模型。
  
  比如，Swift 中的多态存储就比 Rust 的要灵活一些。Swift 里的 `MutableCollection` 会实现一个通过索引来访问元素的 `subscript` 下标方法，但是这个方法几乎可以以任何方式来进行实现并满足这个需求。如果有代码访问了这个 `subscript`，而它又正好是通过直接访问底层内存来实现的话，这个访问就将会发生在原地。但是如果 `subscript` 是以 getter 和 setter 的计算属性方式方式实现的话，访问将会发生在一个临时变量中，getter 和 setter 则会在需要时被调用。因为 Swift 的访问模型是高度词法化的，它保留了在访问的末端运行任意代码的可能性。想象一下，如果我们要实现一个循环，来将这些临时的可变引用添加到一个数组里，我们就需要在循环的每次迭代里都能把任意的代码添加到执行队列里，这样才能在函数操作完数组后进行清理工作。这肯定不会是一个低损耗的抽象！一个在生命周期规则下的，和 Rust 更相似的 `MutableCollection` 接口，需要保证 `subscript` 返回的是一个指向已存在的内存的指针；这样一来，下标就完全不能支持计算方式的实现了。
  
  > 译者注：Swift 的下标访问是一个很有意思的话题。与一般的值变量的复制行为不同，数组下标的访问正是直接的原地访问。但是这是借助于额外的地址器 (Addressors) 来完成的。在数组的下标方法中，并没有返回对应下标元素的值，而是返回了可以获取到元素值的更底层的访问方法 (accessor)。这样一来，写时复制的优化便可以对数组下标访问适用。而那些我们直接返回值的下标方法并不能从中获益，因为它们其实还是通过“计算”来返回下标访问，虽然这个计算本身仅仅只是返回了单个的值。有关更多地址器的问题，Swift 团队也有[详细的文档](https://github.com/apple/swift/blob/master/docs/proposals/Accessors.rst#addressors)进行介绍。

  对于简单的 `struct` 成员，也存在同样的问题。Rust 的生命周期规则中有这样规定：如果你有一个指向 `struct` 的指针，那么你可以创建一个指向该 `struct` 中的一个字段的指针，并且这个新指针会和原来的指针具有同样的生命周期。不过，这条规则不仅假定了字段是确实被存储在内存中的，而且假定了这个字段是被**简单**存储的，也就是说，你可以用一个简单的指针指向它，而且这个指针将满足所指向类型的指针的标准应用二进制接口 (Application Binary Interface, ABI)。这意味着 Rust 不能使用很多内存布局优化手段，比如像是将很多布尔字段打包到一个字节里，或者仅仅是减少某个字段的对齐方式等。我们不愿意将这种保证引入到 Swift 中。

基于上述原因，虽然我们对进一步探索能够承载暂态值更多应用的更复杂的系统抱有理论上的兴趣，我们现在暂时并不打算进行相关的提案。因为这样一个系统所主要包含的是对类型系统的改变，所以我们并不担心这会在长期导致 ABI 稳定上的问题。我们也不用担心这会造成源码不兼容的情况。我们相信，关于这方面任何的增强都可以当作是对我们所提案的特性的扩展和推论来完成。

### 本地暂态量绑定

在 Swift 中，对存储进行抽象，就只能将这个存储通过 `inout` 参数的方式传递，这是一个很蠢的限制。想要一个本地 `inout` 绑定的程序员，可以通过引入一个闭包，并且立即调用这个闭包的方式来轻易地绕开这个限制。这件原本是很简单的事情，不应该要如此麻烦才能达成。

共享值会让这个限制更加明显，用本地的共享值对一个本地的 `let` 值进行替换是一件很有意思的事情：共享值可以避免进行复制，而为此付出的代价是阻止其他的对原来存储的访问。我们不会鼓励程序员在他们的代码中通篇去使用 `shared` 来代替 `let`，特别是优化器通常都能够将复制操作给去除掉。但是，优化器也并不是永远都能移除复制操作，因此 `shared` 这个微优化在一些特定的情形下会很有用。而且，当与不可复制类型打交道时，去除掉正式的复制操作可能在语义上也是必要的。

我们提议直接去除掉这些限制：

```
  inout root = &tree.root

  shared elements = self.queue
```

本地暂态量的初始赋值是必须的，而且它必须是一个存储引用表达式。对于这类值的访问持续到剩余作用域的结束。

> 译者注：也就是说，让 `inout` 和 `shared` 的声明方式能够被一般程序员使用。不过其实对于绝大多数顶层应用的开发者来说，应该是不太用得到这两个声明关键字的。

### 函数参数

函数参数是程序里最重要的对值进行抽象的方式。Swift 现在提供三种方式的参数传递：

- 通过具有所有权的值进行传递。这是一般参数的规则，我们无法显式地指明使用该方式。

- 通过共享的值进行传递。这是对 `nonmutating` 方法的 `self` 参数的规则，我们无法显式地指明使用该方式。

- 通过引用传递。这是对 `inout` 参数和 `mutating` 方法的 `self` 参数的规则。

> 译者注：没错，那些 `nonmutating` 的方法也具有 `self` 参数。(否则你就无法在方法内部使用 `self` 了！)

我们提议，允许我们可以指明使用那些非标准的情况：

- 函数的参数可以被显式指明为 `owned`：

  ```
    func append(_ values: owned [Element]) {
      ...
    }
  ```

  This cannot be combined with `shared` or `inout`.
  
  `owned` 不能和 `shared` 或者 `inout` 一起使用。
  
  它只是对默认情况的一种显式的表达。我们不应该希望用户经常把它们写出来，除非用户正在处理不可复制的类型。

- 函数的参数可以被显式指明为 `shared`。

  ```
    func ==(left: shared String, right: shared String) -> Bool {
      ...
    }
  ```

  This cannot be combined with `owned` or `inout`.
  
  `shared` 不能和 `owned` 或 `inout` 一起使用。
  
  如果函数参数是一个存储引用表达式的话，该存储在调用期间将被作为读取来进行访问。否则，参数表达式将被作为 r-value 来求值，并且临时值在调用中被共享。允许函数参数的临时值被共享是非常重要的，很多函数仅仅是因为自己事实上并不会去拥有参数，它们的参数就将被标记为 `shared`。而其实上，在语义上这些参数被作为对一个已存在的变量的引用时，才是标记 `shared` 的更为重要的情况。举个例子，我们这里将等号操作符的参数标为 `shared`，是因为它们需要在不事先声明的情况下，就能够对不可复制值也进行比较。同时，这也不能妨碍程序员去比较一般的字面值。
  
  和 `inout` 一样，`shared` 是函数类型的一部分。不过与 `inout` 不同，大多数函数兼容性检查 (比如方法重写的检查和函数转换的检查) 在 `shared` 和 `owned` 不匹配时也应该成功。如果一个带有 `owned` 参数的函数被转换 (或是重写) 为了一个 `shared` 参数的函数，参数类型实际上必须是可复制的。

- 方法可以被显式声明为 `consuming`。

  ```
    consuming func moveElements(into collection: inout [Element]) {
      ...
    }
  ```

  这会使 `self` 被当作一个 `owned` 值传入，所以 `consuming` 不能和 `mutating` 或 `nonmutating` 混用。

  在方法内，`self` 依然是一个不可变的绑定值。
  
  > 译者注：这里提出的 `consuming` 实际上是对 `mutating` 的一种更加严谨的细分。如果没有添加相应的约定，那么在使用 `mutating` 时，`self` 的独占性保证只能动态进行。而这也正是 `struct` 中 `mutating` 现在不受程序员待见的原因之一。

### 函数结果

我们在本节的开头进行过一些讨论，想要对 Swift 的词法访问模型进行扩展，让它能支持从函数中返回暂态量并不是一件容易的事。实现这样访问，需要在访问的开始和结束时都执行一些和存储相关的代码。而在一个函数返回后，访问其实就没有进一步执行代码的能力了。

当然了，我们可以返回一个包含暂态量的回调，然后等待调用者使用完这个暂态量后再调用回调，这样我们就能处理暂态量的存储代码了。然而，单单只是这样做还不够，因为被调用者有可能会依赖于它的调用者所做出的保证。举例来说，比如 `struct` 上的一个 `mutating`，它想要返回的是对一个存储属性的 `inout` 引用。想要一切正确，我们不仅要保证在访问属性后方法能够进行清理工作，还要保证绑定在 `self` 上的变量也一直有效。我们真正想要做的是在被调用侧以及调用侧所有有效的作用域内对当前上下文进行维护，并且简单地将暂态量作为参数，在调用侧进入一个新的嵌套的作用域。在编程语言中，这是一个已经被充分理解的情况了，那就是协程 (co-routine)。(因为作用域限制，你也可以将它想象为一个回调函数的语法糖，其中的 `return` 和 `break` 等都按照期望工作。)

事实上，协程可以用来解决很多有关暂态量的问题。我们会在接下来的几个子章节内探索这个问题。

> 译者注：协程的概念可以帮助简化线程调度的问题，也是一个良好的异步编程模型的基础。

### `for` 循环

和三种传递参数的方式相同，我们也可以将对一个序列进行循环的方式分为三种。每种方式都可以用一个 `for` 循环来表达。

#### Consuming 迭代

第一种迭代方式是 Swift 中我们已经很属性的方式了：消耗 (consuming) 迭代，这种迭代中每一步都由一个 `owned` 值来代表。这也是我们对那些值可能是按需生成的任意序列的唯一的迭代方式。对于不可复制类型的集合，这种迭代方式可以让集合最终被结构，同时循环将获取集合中元素的所有权。因为这种方式会取得序列所产生的值的所有权，而且任意一个序列都不能被多次迭代，所以对于 `Sequence` 来说，这是一个 `consuming` 操作。

我们可以显式地将迭代变量声明为 `owned` 来指明这种迭代方式：

```
  for owned employee in company.employees {
    newCompany.employees.append(employee)
  }
```

当非可变的迭代的要求不能被满足的时候，这种方式也应该被默认地使用。(而且，这也是保证源码兼容性所必须的。)

接下来两种方式只对集合有意义，而不适用于任意的序列。

> 译者注：Swift 中，集合 (`Collection`) 一定是序列 (`Sequence`)，但是序列不一定是集合。

#### Non-mutating 迭代

非可变迭代 (non-mutating iteration) 所做的事情是简单地访问集合中的每个元素，并且保持它们完好不变。这样，我们就不需要复制这些元素了；迭代变量可以简单地绑定给一个 `shared` 的值。这就是 `Collection` 上的 `nonmutating` 操作。

我们可以显式地将迭代变量声明为 `shared` 来指明这种迭代方式：

```
  for shared employee in company.employees {
    if !employee.respected { throw CatastrophicHRFailure() }
  }
```

在序列类型满足 `Collection` 时，这种行为是默认行为，因为对于集合来说，这是一种更优化的做法。

```
  for employee in company.employees {
    if !employee.respected { throw CatastrophicHRFailure() }
  }
```

如果序列操作的是一个存储引用表达式的话，在循环持续过程中，存储会被访问。注意，这意味着独占性原则将隐式地保证在迭代期间集合不会被修改。程序可以对操作使用固有函数 (intrinsic function) `copy` 来显式地要求迭代作用在存储的复制值上。

> 译者注：其实我们或多或少已经从 Swift 的值特性和独占性中获取好处了。比如对于一个可变数组，我们可以在迭代它的同时，修改它内容：
>
```swift
var array = [1,2,3]
for v in array {
   let index = array.index(of: v)!
   array.remove(at: index)
}
```
> 这正得益于对于迭代变量的复制，而这种操作在很多其他语言里是难以想象的。不过，这种语义上没有问题的做法却可能在实际中给程序员造成一些困扰。使用明确的标注来规范这种写法确实会是更好的选择。
> 
> 关于固有函数，是指实现由编译器进行处理的那些“内嵌”在语言中的函数。我们会在后面的章节再进行详细说明。


#### Mutating 迭代

一个可变迭代将访问每个元素，并且有可能对元素作出改变。所迭代的变量是一个对元素的 `inout` 引用。这是对 `MutableCollection` 的 `mutating` 操作。

这种方式必须显式地用 `inout` 来声明迭代变量：

```
  for inout employee in company.employees {
    employee.respected = true
  }
```

序列操作的必须是一个存储引用表达式。在循环的持续时间中，存储将会被访问，和上面一种方式一样，这将阻止对于集合的重叠访问。(但是如果集合类型定义的操作是一个非可变操作的话，这条规则便不适用，比如一个引用语义的集合就是如此。)

#### 表达可变和不可变迭代

可变迭代和不可变迭代都要求集合在迭代的每一步创建一个暂态量。在 Swift 中，我们有若干种表达的方式，但是最合理的方式应该是使用协程。因为协程在为调用者产生 (yield) 值的时候不会丢弃自己的执行上下文，所以一个协程产生多个值就是很正常的用法了，这也非常符合循环的基本代码模式。由此产生的一类协程通常被称为生成器 (generator)，这也正是很多种主要语言实现迭代的方式。在 Swift 中，为了也能实现这种模式，我们需要允许对生成器函数进行定义，比如：

```
  mutating generator iterateMutable() -> inout Element {
    var i = startIndex, e = endIndex
    while i != e {
      yield &self[i]
      self.formIndex(after: &i)
    }
  }
```

对于使用者一方，用生成器来实现 `for` 循环的方式是很明显的；不过，如何直接在代码中允许生成器的使用却不那么明显的事情。如上所述，因为逻辑上整个协程必须运行在对原来值访问的作用域中，所以对于协程使用的方式，有一些有趣的限制。对于一般的生成器而言，如果生成器函数返回的确实是某种类型的生成器对象，那么编译器必须确保这个对象不会逃逸出访问范围。这是复杂度的一个重要来源。

### 一般化的访问方法

Swift 现在提供的用来获取属性和下标的工具相当粗糙：基本上只有 `get` 和 `set` 方法。对于性能很关键的任务来说，这些工具是远远不足的，因为它们并不支持直接对值进行访问，而一定会发生复制。标准库中可以使用稍微多一些的工具，可以在特定有限的情况下提供直接的访问，但是它们仍然很弱，而且基于不少原因，我们并不希望将它们暴露给用户。

所有权给我们提供了一个重新审视这个问题的机会，因为 `get` 返回的是一个拥有所有权的值，所以它无法用于那些不可复制类型。访问方法 (getter 或者 setter) 真正需要的是产生一个共享值的能力，而不只是单单能返回值。同样地，想要达成这一目的的一个可行方式是让访问方法能使用某种特殊的协程。和生成器不同，这个协程只能进行一次发生。而且我们没有必要为程序员设计调用它的方式，因为这种协程只会被用在访问方法中。

我们的想法是，不去定义 `get` 和 `set`，而是将在存储声明中定义 `read` 和 `modify`：

```
  var x: String
  var y: String
  var first: String {
    read {
      if x < y { yield x }
      else { yield y }
    }
    modify {
      if x < y { yield &x }
      else { yield &y }
    }
  }
```

一个存储声明必须定义 `get` 或者 `read` (或者定义为存储属性) 中的一个，但是不应该进行同时定义。

如果想要可变的话，存储声明必须再定义 `set` 或者 `modify` 中的一个。不过也可以选择同时定义**两者**，这种情况下 `set` 会被用作赋值，而 `modify` 会被用作更改。这在优化某些复杂的计算属性时会很有用，因为它可以允许更改操作原地进行，而不用强制对首先读取的旧值进行重新赋值。不过，需要特别注意，`modify` 的行为必须和 `get` 和 `set` 的行为相一致。

### 固有函数

#### `move`

Swift 优化器一般会尝试将值进行移动，而不是复制它们，但是强制进行移动也有其意义。正因如此，我们提议加入 `move` 函数。从概念上说，`move` 函数是一个 Swift 标准库的顶层函数：

```
  func move<T>(_ value: T) -> T {
    return value
  }
```

然后，这个函数有一些特定的含义。该函数不能被间接使用，参数表达式必须是某种形式的本地所有的存储：它可以是一个 `let`，一个 `var`，或者是一个 `inout` 的绑定。调用 `move` 函数在语义上等同将当前值从参数变量中移动出来，并将其以表达式制定的类型进行返回。返回的变量在最终初始化分析中将作为未初始化来对待。接下来变量所发生的事情依赖于变量的种类而定：

- `var` 变量将被作为未初始化而简单传回。除非它被赋以新值或者被再次初始化，否则对它的使用都是非法的。

- `inout` 绑定和 `var` 类似，不过它不能在未初始化的情况下离开作用域。或者说，如果程序要离开一个有 `inout` 绑定的作用域的话，程序必须为这个变量赋新的值，而不论它是以何种方式离开作用域 (包括抛出错误的时候)。将 `inout` 暂时作为未定义变量的安全性是由独占性原则所保证的。

- `let` 变量不能被再次初始化，所以它不能再被使用。

这对于现在的最终初始化分析是一个直接的补充，它能确保在使用一个本地变量之前，它总是被初始化过的。

#### `copy`

`copy` 是 Swift 标准库中的一个顶层函数：

```
  func copy<T>(_ value: T) -> T {
    return value
  }
```

参数必须是一个存储引用表达式。函数的语义和上面的代码一致：参数值会被返回。该函数的意义如下：

- 它可以阻止语法上的特殊转换。举例来说，我们上面讨论过，如果 `shared` 参数是一个存储引用，那么存储在调用期间是被访问的。程序员可以通过事前在存储引用上调用 `copy` 来阻止这种访问，并且强制复制操作在函数调用前完成。

- 对于那些阻止隐式复制的类型来说，这是必须的。我们会对不可复制类型进行进一步叙述。

#### `endScope`

`endScope` 是 Swift 标准库中的顶层函数：

```
  func endScope<T>(_ value: T) -> () {}
```

参数必须是一个引用本地 `let`，`var` 或者独立的 (非参数，非循环) `inout` 或者 `shared` 声明。如果参数是 `let` 或者 `var`，则变量会被立即销毁。如果参数是 `inout` 或者 `shared`，则访问将立即终止。

最终初始化分析必须保证声明在这个调用后没有再被使用。如果存储是一个被逃逸闭包捕获的 `var` 的话，应该给出错误。

对于想要在控制流到达作用域结尾前就想要终止访问的情形来说，这很有用。同样地，对销毁值时的微优化也能起到作用。

`enScope` 保证在调用的时候输入的值是被销毁的，或者对它的访问已经结束。不过它没有承诺这些事情确实发生在这个时间点：具体的实现仍然可以更早地结束它们。

### 透镜 (Lenses)

现在，Swift 中所有的存储引用表达式都是**具体**的：每一个组件都静态地对应一种存储声明。在社区中，大家在允许程序对存储进行抽象这件事上一致兴趣盎然，比如说：

```
  let prop = Widget.weight
```

这里 `prop` 会是一个对 `weight` 属性的抽象引用，它的类型是 `(Widget) -> Double`。

> 译者注：对于类型上的方法来说，这种透镜抽象是一直存在的 - 因为方法不存在所有权的内存问题。

这个特性和所有权模型有关，因为一个一般的函数的结果一定是一个 `owned` 的值：不会是 `shared`，也不是可变值。这意味着，上述这种透镜抽象只能抽象**读操作**，而不能对应**写操作**，而且我们只能为可复制的属性创建这种抽象。这也意味着使用透镜的代码会比使用具体存储引用的代码需要更多的复制。

设想，要是透镜抽象不是简单的函数，而是它们各自类型的值。那么透镜的使用将会变成一个对静态未知成员进行访问的抽象的存储引用表达式。这就需要语言的实现能够执行某种程度的动态访问。然而，访问未知的属性和访问实现未知的已知属性有几乎一样的问题；也就是说，为了实现泛型和还原类型，语言已经需要做类似的事情了。

总体来说，只要我们有所有权模型，这样的特性就正好可以符合我们的模型。

## 不可复制的类型

不可复制的类型在很多高级的情况中会十分有用。比如，它们可以被用来高效地表达唯一的所有权。它们也可以用来表达一些含有像是原子类型这样的某种独立标识的值。它们还可以被用作一种正式的机制，来鼓励代码能够更高效地和那些复制起来开销很大的类型 (比如很大的) 一起使用。它们之间统一的主题是，我们不想类型被隐式地复制。

Swift 中处理不可复制类型的复杂度主要有两个来源：

- 语言必须提供能将值进行移动和共享，而不强制进行复制的工具。我们已经对这些工具进行了提案，因为它们对优化可复制类型的使用也同等重要。

- 泛型系统必须能够表达不可复制的类型，同时不引入大量的源码兼容性问题，也不需要强制所有人使用不可复制类型。

如果不是因为这两个原因的话，这个特性本身是很小的。就只需要用我们上面提到的 `move` 固有函数那样，隐式地使用移动来代替复制，并且在遇到无法适用的情况下给出诊断信息即可。

### `moveonly` 上下文

不过，泛型确实会是一个问题。在 Swift 中，最直白的为可复制特性建模的方式无非就是添加一个 `Copyable` 协议，让那些可以被复制的类型遵守这个协议。这样一来，不加限制的类型参数 `T` 就无法被假设为可复制类型。不过，这么做对源码兼容性和可用性来说都会是巨大的灾难，而且我们也不想让程序员在首次被介绍使用泛型代码的时候就去操心那些不可复制类型的问题。

另外，我们也不想让类型需要显式地被声明为支持 `Copyable`。对复制的支持应该是默认的。

所以，逻辑上来说解决的方式是，维持现在所有类型都是可复制类型的默认假设，然后允许上下文选择将这个假设关掉。我们将这些上下文叫做 `moveonly` 上下文。在一个 `moveonly` 上下文中词法嵌套的所有上下文也都隐式地成为 `moveonly`。

一个类型可以提供 `moveonly` 上下文：

```
  moveonly struct Array<Element> {
    // Element and Array<Element> are not assumed to be copyable here
  }
```

这将阻止在该类型声明，它的泛型参数 (如果有的话)，以及它们在继承链上所关联的类型上进行 `Copyable` 假设。

扩展也可以提供 `moveonly` 上下文：

```
  moveonly extension Array {
    // Element and Array<Element> are not assumed to be copyable here
  }
```

不过使用带有条件的协议遵守时，类型可以声明为条件可复制：

```
  moveonly extension Array: Copyable where Element: Copyable {
    ...
  }
```

不论是在约束条件里满足还是直接满足 `Copyable`，它都会是一个类型的继承特性，并且一定要在定义该类型的同一个模块中进行声明。(或者有可能的话，应该在同一个文件中进行声明。)

对于一个类型的非 `moveonly` 的扩展，将会把可复制性的假设重新引入这个类型及其泛型参数中。这么做的目的是为了标准库中的类型能够在不打破现有扩展的兼容性的同时，添加对不可复制元素的支持。如果一个类型没有进行任何的 `Copyable` 声明，那么为它添加一个非 `moveonly` 的扩展将会发生错误。

> 译者注：这里的意思是，针对那些 `moveonly` 定义的类型，我们不能为它随意添加非 `moveonly` 的扩展。这是显而易见的，否则就会发生复制特性的冲突。而对于那些非 `moveonly` 的类型 (因为它们是隐式默认支持复制，或者说满足 `Copyable` 的)，以及在条件约束下满足 `Copyable` 的情况来说，添加非 `moveonly` 扩展是没有问题的。

函数也可以定义一个 `moveonly` 上下文：

```
  extension Array {
    moveonly func report<U>(_ u: U)
  }
```

这将会使任何新的泛型参数和它们的继承关联类型上的复制假设无效。

很多关于 `moveonly` 上下文的细节我们扔在考虑之中。关于这个问题，在我们最终寻找到正确的设计之前，还需要很多的进行语言实现的经验。

我们正在考虑的一种可能性是，对于可复制类型的值，`moveonly` 上下文也将会取消其可复制假设。对于那些需要特别注意复制操作的代码来说，这会提供一种重要的优化工具。

### 不可复制类型的 `deinit`

对那些定义为 `moveonly` 的不遵守 (也不条件遵守) `Copyable` 的值类型，可以为其定义一个 `deinit` 方法。注意，`deinit` 必须被定义在类型的主定义域内，而不能定义在扩展中。

在值不再被需要时，`deinit` 将会被调用以销毁这个值。这让不可复制类型可以被用来表达对于资源的唯一所有权。比如说，这里有一个简单的处理文件的类型，它保证了值被销毁时，文件句柄一定会被关闭：

```
  moveonly struct File {
    var descriptor: Int32

    init(filename: String) throws {
      descriptor = Darwin.open(filename, O_RDONLY)

      // 在 `init` 里任何非正常退出都会阻止 deinit 被调用
      if descriptor == -1 { throw ... }
    }

    deinit {
      _ = Darwin.close(descriptor)
    }

    consuming func close() throws {
      if Darwin.fsync(descriptor) != 0 { throw ... }

      // 这是一个 consuming 函数，所以它拥有对自己的所有权。
      // 其他的任何方式都不会对 self 产生消耗，所以函数将在
      // 退出时通过调用 deinit 进行销毁。
      // 而 deinit 将会通过描述符实际关闭文件句柄。
    }
  }
```

Swift 对值的销毁 (以及对 `deinit` 的调用) 发生在一个值被最后使用后，以及正式解构的时间点前的期间内。不过这个定义中对于“使用”的定义暂时还没有完全决定。

如果这个值类型是一个 `struct`，那么 `deinit` 中 `self` 只能被用来引用类型的存储属性。`self` 的存储属性会被当作本地 `let` 常量被看待，并用于最终初始化分析；也就是说，它们是属于 `deinit` 方法，并且可以被移出去的。

如果值类型是一个 `enum` 的话，`deinit` 里的 `self` 只能被当作 `switch` 的操作数来使用。在 `switch` 内，任何一个用来初始化对应绑定的关联值，都拥有对这些值的所有权。这样的 `switch` 会使 `self` 处于未初始化状态。

### 显式可复制类型

在不可复制类型里，还有一种我们在探索的想法，那就是将一个类型声明为不可被隐式复制。比如，一个很大的结构体可以被正式地进行复制，但是如果不必要地对它进行复制的话，就会对性能产生过大的影响。这样的类型应该需要遵守 `Copyable`，而且它应该在调用 `copy` 函数时请求一份复制。不过，编译器应当像在处理不可复制类型那样，在任何隐式复制发生时给出诊断信息。

## 实现的优先级

这篇文档陈列了很多工作，我们可以将其总结如下：

- 强制独占性原则：

  - 静态强制
  - 动态强制
  - 动态强制的优化

- 新的标注和声明：

  - `shared` 参数
  - `consuming` 方法
  - 本地 `shared` 和 `inout` 声明

- 新的固有函数和它们的区别：

  - `move` 函数及其关联的影响
  - `endScope` 函数及其关联的影响

- 协程特性：

  - 通用的访问方法
  - 生成器

- 不可复制类型

  - 未来的设计工作
  - 不可复制类型的区别
  - `moveonly` 上下文

在接下来的版本中，最主要的目的是 ABI 稳定。对于这些特性的优先级划分和分析必须以它们对 ABI 的影响为中心。在将这一点纳入考虑后，我们主要对 ABI 方面有如下思考：

独占性原则将会改变对参数作出的保证，因此它将影响 ABI。我们必须在 ABI 锁定之前将这条规则纳入到语言中，否则我们将永远失去改变这个保守假设的机会。不过，除非我们打算将一部分工作放到运行时去做，否则具体的实现独占性原则的方式并不会对 ABI 产生影响。况且将部分工作放到运行时并不是必要的，它在未来的发布版本中也可以被改变。(另外需要说明，在技术上独占性原则可能给优化器造成重大的影响，但是这应该是一个普通的项目进程上的考虑，而不会影响到 ABI。)

> 译者注：Swift 的 ABI 稳定是一个提了有两年的议题了。现在看来，Swift 4 中 ABI 稳定依然无法达成，也就是说不同 Swift 编译器编译出的二进制并不能互相通用 (举例来说，就是新版本 Swift 的 app 不能调用旧版本的 Swift 框架)。如果没有 ABI 稳定，Swift app 就还是必须包含 Swift 运行库的复制，我们也不可能使用二进制的框架。Apple 当前内部 app 和自己的框架几乎都不是 Swift 版本的，也在很大程度上受到 Swift ABI 稳定性的限制。

标准库会积极地在参数上适配所有权标注。那些标注肯定会影响这些库的 ABI。库开发者需要时间来进行适配，更重要的是，它们需要一些方式来验证标注是有用的。不幸的是，用来验证的最好的方法是实现不可复制的类型，而这在优先级列表上是排名很低的任务。

通用的访问方法所需要的工作包括，将“最通用”的属性和下标标准访问方法从 `get`/`set`/`materializeForSet` 变更为 `read`/`set`/`modify`。这对所有的多态属性和下标访问都会带来 ABI 影响，所以它也必须先做。不过，这个 ABI 的变更可以在不实际将协程方式的访问方法引入 Swift 的前提下完成。重要的只是保证我们所使用的 ABI 在今后能够满足协程的要求。

生成器部分的工作可能会改变核心的集合协议。显然这会影响 ABI。和通用化的访问方法不同，我们绝对需要实现生成器，来让 ABI 满足我们的需求。

不可复制的类型和算法只会影响到标准库中对它们进行适配的范围的 ABI。如果库想要在标准的集合中对它们进行适配和扩展的话，就必须发生在 ABI 稳定之前。

新的本地声明和固有函数不会影响 ABI。(和大多数情况一下，影响最少的工作往往也是最简单的工作。)

看起来，想在标准库中适配所有权和不可复制类型，会有很多工作要做，但是这对不可复制类型的可用性来说十分重要。如果我们不能创建一个包含不可复制类型的 `Array` 的话，对语言来说，这将会是非常大的限制。

> 译者注：长求总？好的，总结一下全文，
> 这个提案提出在今后的 Swift 版本 (极大可能是 Swift 4) 中引入如下变化：
> 
>   - 强制的独占性原则，违反该原则的代码将发生错误：
>     - 如果在静态能检出违反独占性原则，则给出编译错误
>     - 如果在运行时动态检出违反独占性原则，则在违反时让代码崩溃
>     - 对于不安全指针的情况，独占性原则行为将是未定义
>   - 添加 `shared`，`owned` 和 `consuming` 关键字
>     - `shared` 用在参数或者声明上，表示不获取值的所有权，以此避免不必要的值复制。
>     - `owned` 和 `consuming` 分别用在变量和函数上，表示非 `shared` 方式的调用。
>   - 增强访问方法，在 `get` 和 `set` 的基础上，添加 `read` 和 `modify` 方法。其中 `read` 对应 `shared` 参数，`modify` 对应 `inout`。
>   - 新的迭代方式，可以使用 `shared` 或 `inout` 来规定迭代变量的所有权。作为衍生，我们需要协程的方式进行实现。也就是说，可能会在语言中引入原生的生成器 (generator)。这也是进一步进行异步编程的基础，想要了解更多这方面内容，可以参看我的[另一篇博文](https://onevcat.com/2016/12/concurrency/)。
>   - 引入一系列所有权有关的固有函数，如 `move`，`copy` 和 `endScope`。它们为 Swift 的高级用户提供自行进行所有权管理的可能性。
>   - 对于不可复制的类型，引入 `moveonly` 来去除默认的可复制假设。
> 
> 另外，这篇宣言只是一些基础的设想和讨论。也就是说，有些细节，比如关键字的名字或者具体的实现方式还可能有变。不过，Swift 团队想要明确值的所有权的问题的方向应该是不变的。
> 
> 通过这些新的手段和方式，我们可以厘清所有权，避免复制和进行相关优化，但是这并不意味着作为 Swift 的最终用户的程序员在开发时的写法会发生翻天覆地的变化。不过，理解 Swift 中值的所有权变化，可以让我们对这门语言的设计有更深的了解和思考。
> 
> 如果你暂时无法理解和接受这些内容，在 Swift 4 发布后，我们应该可以获得更多面向语言的“使用者”的信息和例子，来帮助我们最终作出决定。


