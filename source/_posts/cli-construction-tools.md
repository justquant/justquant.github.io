---
title: 使用scopt解析Spark参数
date: 2016-11-21 11:00:20
tags: [scala]
---

最近在开发一个Spark程序，需要解析自定义的参数。经过一番搜索，选定了两个工具：

- [Apache Commons CLI](https://commons.apache.org/proper/commons-cli/)
- [Scopt, Simple scala command line options parsing](https://github.com/scopt/scopt)

从官方介绍来看，Apache Commons CLI功能比较强大，可以支持丰富的语法。但最终我们还是选定了scopt，主要是因为它是scala写的，而且我们只需要一个简单的解析器。


## 第一步：将scopt添加到sbt工程的依赖

```
libraryDependencies += "com.github.scopt" %% "scopt" % "3.5.0"
```

## 第二步：定义一个 case class

一般来说，要给每一个参数赋予一个默认值。因为scopt在parse参数的时候，是从一个空的对象开始的。是使用Immutable的方式，每parse一个参数，都是在原来的case class的对象上复制一个新的对象，并赋予该参数对应的值。

```
case class Configuration(input:String = "", output:String = "", number:Int = 0, other:String = "")
```

## 第三步：定义parser，这里一般使用scopt.OptionParser。这样可以指定一个参数是必须的还是可选的。
```
object ScoptDemo {

  def main(args: Array[String]): Unit = {
    val parser = new scopt.OptionParser[Configuration]("scopt demo") {
      head("scopt","3.x")

      opt[String]('i',"input").required().action((x,config) => config.copy(input = x)).text("input directory")

      opt[String]('o', "output").required().action((x,config) => config.copy(output = x)).text("output directory")

      opt[String]('n', "num").optional().action((x,config) => config.copy(number = x.toInt)).text("number of inputs")

      opt[String]('t',"other").optional().action((x,config) => config.copy(other = x)).text("others")
    }

    parser.parse(args, Configuration()) match {
      case Some(configuration) => println(configuration)
      case None => println("cannot parse args")
    }
  }

}
```

以下是直接执行该程序（不传入任何参数）的错误提示：
```
cannot parse args
Error: Missing option --input
Error: Missing option --output
scopt 3.x
Usage: scopt demo [options]

  -i, --input <value>   input directory
  -o, --output <value>  output directory
  -n, --num <value>     number of inputs
  -t, --other <value>   others
```

而执行传入参数列表：-i input1 -o output2 -n 10 -t "no others"，得到的输出为：

Configuration(input1,output2,10,no others)

可见scopt还是非常友好的。scopt还支持其它的功能，例如：

- 对输入的参数进行校验
- 支持command。command还可以自带自己的参数


