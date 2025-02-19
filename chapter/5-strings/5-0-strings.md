# 5 字符串

> **本章概要**
> * 理解 Go 中符文的基本概念
> * 防止字符串迭代和修剪的常见错误
> * 避免因字符串连接或无用转换而导致的低效代码
> * 使用子字符串避免内存泄漏

在 Go 中，字符串是一种不可变的数据结构，相当于：

* 指向不可变字节切片的指针
* 此数组中的总字节数

我们将在本章中看到 Go 有一种非常独特的方式来处理字符串，并引入了一个名为符文(runes)的概念。这个概念对于理解是必不可少的，并且可能会使新手感到困惑。一旦我们了解了字符串的管理方式，我们就可以在迭代字符串时避免常见错误。我们还将看到 Go 开发人员在使用或生成字符串时所犯的常见错误。此外，我们会看到有时我们可以直接使用 `[]byte`，避免额外的分配。最后，我们将看到如何避免从子字符串中创建泄漏的常见错误。本章的主要目标是通过介绍常见的字符串错误来帮助您理解 Go 中字符串的工作原理。
