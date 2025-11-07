# 前置内容

## 前言

“你能描述一个执行线程导致死锁的情况吗？”我的面试官问道。在我给出正确答案后，他进一步追问：“在那个情况下，你会怎么做来确保代码避免死锁？”幸运的是，我也知道解决方案。面试官接着给我展示了一些代码，并询问我是否能发现其中的任何问题。代码中存在一个糟糕的竞争条件，我指出了这一点，并提出了解决问题的方法。

这个问题是在我第三次也是最后一次面试伦敦一家国际科技公司核心后端开发职位时提出的。在这个职位上，我接触到了编程中最具挑战性的问题——这些问题要求我提高开发并发、低延迟和高吞吐量服务的技能。那是在 15 年前。

在我 20 多年的技术生涯中，许多情况都发生了变化：开发者现在可以随时随地工作，计算机语言已经发展得能够模拟更复杂的业务，而极客们因为现在运营着大型科技公司而变得酷炫。然而，一些方面保持不变：程序员总是苦于给变量命名，许多问题可以通过关闭系统然后重新启动来解决，而并发编程技能仍然供不应求。

科技行业缺乏擅长并发的程序员，因为并发编程被认为是一项极具挑战性的任务。许多开发者甚至害怕使用并发编程来解决问题。在科技行业中，这种看法是，这是一个高级话题，仅限于硬核计算机极客。这有很多原因。开发者不熟悉用于管理并发的概念和工具，有时他们未能认识到并发如何被程序化地建模。这本书是我试图解决这个问题，并以一种无痛的方式解释并发编程的尝试。

## 致谢

我对那些在本书写作过程中提供重大贡献的人表示感谢。

首先，我想感谢 Joe Cordina，他在我学习初期就向我介绍了并发编程，激发了我对这个主题的兴趣。

我还想感谢 Miguel David，是他提出了使用 Go 语言发布材料的想法。

我要感谢 Russ Cox 为我提供了关于 Go 语言历史的宝贵指导。

特别感谢开发编辑 Becky Whitney，技术校对员 Nicolas Modrzyk，以及技术编辑 Steven Jenkins，他是一家全球金融服务公司的资深工程师，曾设计、构建并支持从小型初创公司到全球企业的系统。他认为 Go 语言对于高效构建和交付可扩展的解决方案非常有用。他们的专业精神、专业知识和奉献精神确保了这本书达到了最高的质量标准。

致所有审稿人：Aditya Sharma、Alessandro Campeis、Andreas Schroepfer、Arun Saha、Ashish Kumar Pani、Borko Djurkovic、Cameron Singe、Christopher Bailey、Clifford Thurber、David Ong Da Wei、Diego Stamigni、Emmanouil Chardalas、Germano Rizzo、Giuseppe Denora、Gowtham Sadasivam、Gregor Zurowski、Jasmeet Singh、Joel Holmes、Jonathan Reeves、Keith Kim、Kent Spillner、Kévin Etienne、Manuel Rubio、Manzur Mukhitdinov、Martin Czygan、Mattia Di Gangi、Miki Tebeka、Mouhamed Klank、Nathan B. Crocker、Nathan Davies、Nick Rakochy、Nicolas Modrzyk、Nigel V. Thomas、Phani Kumar Yadavilli、Rahul Modpur、Sam Zaydel、Samson Desta、Sanket Naik、Satadru Roy、Serge Simon、Serge Smertin、Sergio Britos Arévalo、Slavomir Furman、Steve Prior 和 Vinicios Henrique Wentz，你们的建议帮助使这本书变得更好。

最后，我想感谢 Vanessa、Luis 和 Oliver 在整个项目中的坚定不移的支持。他们的鼓励和热情让我保持动力，并激励我发挥最佳水平。

## 关于这本书

《用 Go 学习并发编程》是为了帮助开发者通过更高级的并发编程来提高他们的编程技能。选择 Go 作为展示示例的语言，因为它提供了广泛的工具来完全探索这个并发编程的世界。在 Go 中，这些工具非常直观且易于掌握，让我们能够专注于并发编程的原则和最佳实践。

在阅读这本书之后，你将能够

+   使用并发编程编写响应迅速、性能高和可扩展的软件

+   掌握并行计算的优势、局限性和特性

+   区分内存共享和消息传递

+   利用 goroutines、互斥锁、读写锁、等待组、通道和条件变量——并且，此外，了解如何构建这些工具

+   在处理并发执行时识别需要注意的典型错误

+   通过更高级的多线程主题提高你在 Go 中的编程技能

### 应该阅读这本书的人

这本书是为那些已经有一定编程经验并希望了解并发编程的读者所写。本书假设读者没有并发编程的先验知识。尽管理想的读者已经有一定使用 Go 或其他 C 语法类似语言的经验，但本书也适合来自任何语言的开发者——如果花些时间学习 Go 的语法。

并发编程为你的编程增加了另一个维度：程序不再是一系列依次执行的指令。这使得它成为一个具有挑战性的主题，需要你以不同的方式思考程序。因此，虽然已经精通 Go 很重要，但拥有好奇心和动力更为重要。

本书不专注于解释 Go 的语法和特性，而是使用 Go 来展示并发原理和技术。这些技术中的大多数可以应用于其他语言。有关 Go 教程和文档，请参阅[`go.dev/learn`](https://go.dev/learn/)。

### 本书组织结构：路线图

本书分为三部分，共 12 章。第一部分介绍了并发编程的基础以及使用内存共享进行通信：

+   第一章介绍了并发编程，并讨论了并行执行的一些定律。

+   第二章讨论了我们可以建模并发的方式以及操作系统和 Go 运行时提供的抽象。本章还比较了并发和并行性。

+   第三章讨论了使用内存共享进行线程间通信，并介绍了竞态条件。

+   第四章探讨了不同类型的互斥锁作为解决某些竞态条件的方法。它还展示了如何实现基本的读者-写者锁。

+   第五章展示了如何使用条件变量和信号量来同步并发执行。本章还描述了如何从头开始构建信号量，并改进上一章中开发的读者-写者锁。

+   第六章演示了如何构建和使用更复杂的同步机制，例如等待组和屏障。

第二部分讨论了多个执行如何通过消息传递而不是内存共享进行通信：

+   第七章描述了使用 Go 的通道进行消息传递。本章展示了通道可以以各种方式使用，并说明了通道是如何建立在内存共享和同步原语之上的。

+   第八章解释了如何使用 Go 的`select`语句组合多个通道。此外，本章还提供了一些在选择开发并发程序时使用内存共享与消息传递的指南。

+   第九章提供了重用常见消息传递模式的示例和最佳实践。本章还展示了在语言（如 Go）中通道是一等对象时的灵活性。

第三部分探讨了常见的并发模式和一些更高级的主题：

+   第十章列出了分解问题的技术，以便通过使用并发编程使程序运行得更高效。

+   第十一章说明了在存在并发的情况下，死锁情况如何发展，并描述了避免这些情况的各种技术。

+   第十二章处理互斥锁的内部结构。它解释了互斥锁如何在内核和用户空间中实现。

### 如何阅读本书

没有并发经验的开发者应将本书视为一次旅程，从第一章开始，一直读到结尾。每一章都教授新的技能和技术，这些技能和技术建立在之前章节获得的知识之上。

对于已经有一定并发经验的开发者来说，可以阅读第一章和第二章，作为操作系统如何模拟并发的复习，然后决定是否跳过一些更高级的主题。例如，已经熟悉竞态条件和互斥锁的读者可能会选择在第五章继续学习条件变量。

### 关于代码

书中列表中的源代码使用`固定宽度字体`，以区别于文档的其他部分，Go 中的关键字设置为**`粗体`**。许多列表旁边都有代码注释，突出重要概念。

要下载本书中的所有源代码，包括练习解答，请访问[`github.com/cutajarj/ConcurrentProgrammingWithGo`](https://github.com/cutajarj/ConcurrentProgrammingWithGo)。书中示例的完整代码也可以从 Manning 网站[`www.manning.com/books/learn-concurrent-programming-with-go`](https://www.manning.com/books/learn-concurrent-programming-with-go)下载。您还可以从本书的 liveBook（在线）版本中获取可执行的代码片段[`livebook.manning.com/book/learn-concurrent-programming-with-go`](https://livebook.manning.com/book/learn-concurrent-programming-with-go)。

本书中的源代码需要 Go 编译器，可以从[`go.dev/doc/install`](https://go.dev/doc/install)下载。请注意，本书中的一些源代码在 Go 的在线沙盒中可能无法正确运行——沙盒被阻止执行某些操作，例如打开网络连接。

### liveBook 讨论论坛

购买《用 Go 学习并发编程》包括对 Manning 的在线阅读平台 liveBook 的免费访问。使用 liveBook 的独特讨论功能，您可以在全球范围内或对特定章节或段落附加评论。为自己做笔记、提问和回答技术问题以及从作者和其他用户那里获得帮助都非常简单。要访问论坛，请访问[`livebook.manning.com/book/learn-concurrent-programming-with-go/discussion`](https://livebook.manning.com/book/learn-concurrent-programming-with-go/discussion)。您还可以在[`livebook.manning.com/discussion`](https://livebook.manning.com/discussion)了解更多关于 Manning 论坛和行为准则的信息。

Manning 对我们读者的承诺是提供一个场所，让读者之间以及读者与作者之间可以进行有意义的对话。这不是对作者参与特定数量活动的承诺，作者对论坛的贡献仍然是自愿的（且未付费）。我们建议您尝试向作者提出一些挑战性的问题，以免他的兴趣转移！只要本书仍在印刷中，论坛和先前讨论的存档将从出版社的网站提供。

## 关于作者

![图片](img/Cutajar.png)

**詹姆斯·库塔贾尔**是一位对可扩展、高性能计算和分布式算法感兴趣的软件开发者。他在多个行业中从事技术工作已超过 20 年。在其职业生涯中，詹姆斯一直是开源贡献者、博主([`www.cutajarjames.com`](http://www.cutajarjames.com))、技术布道者、Udemy 讲师和作者。当他不编写软件时，他喜欢骑摩托车、冲浪、潜水和驾驶轻型飞机。詹姆斯出生在马耳他，在伦敦生活了近十年，现在在葡萄牙生活和工作。

## 关于封面插图

《用 Go 学习并发编程》封面上的图像是“Homme Tschoukotske”，或“楚科奇人”，取自雅克·格拉塞·德·圣索沃尔的收藏，1788 年出版。每一幅插图都是手工精细绘制和着色的。

在那些日子里，人们通过他们的着装很容易就能识别出他们住在哪里以及他们的职业或社会地位。曼宁通过基于几个世纪前丰富多样的地区文化的书封面来庆祝计算机行业的创新和主动性，这些文化通过如这一系列图片的图片被重新带回生活。
