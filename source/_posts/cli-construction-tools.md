---
title: CLI 参数解析器
date: 2016-11-21 11:00:20
tags: [Tool]
---

最近在开发一个Spark程序，需要解析自定义的参数。经过一番搜索，选定了两个工具：

- [Apache Commons CLI](https://commons.apache.org/proper/commons-cli/)
- [Scopt, Simple scala command line options parsing](https://github.com/scopt/scopt)

从官方介绍来看，Apache Commons CLI功能比较强大，可以支持丰富的语法。但最终我们还是选定了SCOPT，主要是因为它是scala写的，而且我们只需要一个简单的解析器。

这边文章主要介绍SCOPT。Apache Commons CLI请参看其官网。



