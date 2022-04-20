
---
title: Scala学习Part2
date: 2021-09-07
comments: true #是否可评论
toc: true #是否显示文章目录
categories: "学习" #分类
tags:   #标签
    - Scala
---
#### 文件和正则表达式
Source.fromFile(...).getLines.toArray输出文件中的所有行。
Source.fromFile(...).mkString以字符串的形式输出文件内容。
将字符串转成数字，可以使用toInt or toDouble的方式。
使用Java的PrintWriter来写入文本文件。
"regex".r是一个Regex对象。
如果你的正则表达式中包含反斜杠和引号的话，用"""..."""。
如果正则模式包含分组，你可以用下面的语法来提取内容:
for(regex(v1,v2...vn) <- string>)
```scala
import scala.io.Source
val source=Source.fromFile("myfile.txt","UTF-8")
val lineIterator = source.getLines
val source1 = Source.fromURL("http://www.baidu.com","UTF-8")
val source2 = Source.fromString("hello,world!")
//close source after you use
```

##### 进程控制 
scala.sys.process包提供了用于与shell程序交互的工具。你可以使用scala编写shell脚本，
利用scala提供的威力。
```scala
import sys.process._
"ls -al.."!
//这里面完成了一个字符串到processBuilder对象的隐式转换
```
！返回的结果就是被执行程序的返回值，程序执行成功的话就是0，否则就显示返回的非0值。
如果使用!! 则就会以字符串的形式返回执行的结果。

##### 正则表达式
scala.util.matching.Regex类是scala中的正则。
```scala
val numPattern = "[0-9]+".r
for( matchString <- numPattern.findAllIn("99 bottles, 98 bottles"))
println(matchString)

val numitemPattern = "([0-9]+) ([a-z]+)".r
val numitemPattern(num, item) = "99 bottles"
```

#### 特质
scala中的特质类似java中的接口。
特质中未被实现的方法默认就是抽象方法。
```scala
trait Logger {
    def log(msg: String) //这是个抽象方法
}
```
scala中的特定不要求所有的方法都必须是抽象的，这点在java8以后也支持了，java8支持接口中含有
默认的实现。
缺少构造器参数是特质与类之间唯一的技术差别。除此之外，特质可以具备类的所有物质，比如具体的和抽象的
字段，以及超类。

#### 操作符练习
提供操作符用于构造html表格。例如：
Table() | "java" | "scala" || "go" | "rust" ||"jvm" | ".net"
```scala
package org.example

class Table {

  var s: String = ""

  def |(str: String): Table = {
    this.s = this.s + "<td>" + str + "</td>"
    this
  }

  def ||(str: String): Table = {
    this.s = this.s + "</tr><td>" + str + "</td>"
    this
  }

  override def toString(): String = {
    "<table><tr>" + this.s + "</tr><table>"
  }
}

object Table {
  def apply(): Table = {
    new Table()
  }
  def main(args :Array[String]) = {
    val str = Table() | "java" | "scala" || "go" | "rust" ||"jvm" | ".net"
    println(str)
  }
}

```