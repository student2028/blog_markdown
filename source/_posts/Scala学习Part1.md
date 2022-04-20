---
title: Scala学习Part1
date: 2021-09-07
comments: true #是否可评论
toc: true #是否显示文章目录
categories: "学习" #分类
tags:   #标签
    - Scala
---
#### 基础语法
1. 条件表达式
   在scala中if/else表达式有值，这个值就是跟在if or else之后表达式的值。
   在scala中，每个表达式都有一个类型。没有返回值的表达式是Unit类型，类似java中的void。
   ```scala
   val s = if(x > 0) 1 else -1
   ```
2. 块表达式和赋值
   {}块的值取决于最后一个表达式。
   在scala中，赋值动作本身是没有值的，它们的值是Unit类型。
   ```scala
   x = y = 1 //不要这样做
   ```
3. 循环
   for (i <- 1 to n )
   for ( i <- 1 to 10
         j <- i to 10
   )
   Scala中没有提供 break or continue语句来退出循环。
   但提供了一个scala.util.control.Breaks._中的break可以用来退出循环。
   val vec = for(i <- 1 to 10 ) yield i % 3
   这种情况叫for推导式。
4. 默认参数和带名参数和变长参数
   ```scala
   //默认参数和带名参数
   def decorate(str:String, left:String = "[" , right:String ="]") = left + str + right
   //变长参数
   def sum(args: Int*) = {
       var result = 0
       for ( arg <- args )
          result += arg
       result
   }
   val s = sum(1,2,3)
   // val s = sum(1 to 5 ) error
   val s2 = sum( 1 to 5 :_*) //将1 to 5 当作参数序列处理
   def recursiveSum(args : Int*) : Int = {
       if (args.length == 0 ) 0
       else args.head +  recursiveSum(args.tail :_* )
   }
   ```
   使用的时候，可以调用参数名：
   decorate(left="<<<", str="Hello" , right=">>>")


5. 懒值 lazy val
   当val 被声明为lazy时，它的初始化将被推迟，直到我们首次对它取值。
   lazy val words = scala.io.Source.fromFile("path/to/file.txt").mkString
   如果我们不访问words,那么文件并不会被真正的读取。

6. 异常捕获
   ```scala
   try{
       process(new URL("http://www.baidu.com"))
   } catch{
       case _: MalformatURLException => println("bad url:" + url)
       case ex: Exception => ex.printStackTrace()
   } finally{
       /do something
   }
   ```
 #### 数组相关操作
 若长度固定可以使用Array,若长度有变化可以使用ArrayBuffer。
 使用()来访问元素。
 使用for(ele <- arr)来遍历元素。
 在JVM中，scala中的Array以Java的数组方式实现。
 类似Java中的ArrayList，Scala中有ArrayBuffer。
 ```scala
    val nums=new Array[Int](10)
    val s = Array("Hello", "world")
    import scala.collection.mutable.ArrayBuffer
    val b = ArrayBuffer[Int]()
    //b.insert(2,6)
    //b.remove(2,6)
 ```

 #### 映射和元组
 ```scala
 //构造的不可变的Map[String,Int]
 val scores = Map("alice" -> 10, "bob" -> 3 , "Cindy" -> 8)
// val scores = new scala.collection.mutable.HashMap[String,Int]
   for ((k,v) <- scores )  println(k)
 ```
 如果想按插入的顺序来访问所有键，使用LinkedHashMap,
 val months = scala.collection.mutable.LinkedHashMap("Jan" -> 1 ....)

 #### 类
 用@BeanProperty注解来生成JavaBeans的getXXX/setXXX方法。
 scala中，类并不声明为public,scala源文件可以包含多个类，所有这些类都具有公有可见性。
 Java的Beans规范把java属性定义为一对getFoo/setFoo方法。
 ```scala
 import scala.reflect.BeanProperty
 class Person {
     @BeanProperty var name: String = _
     def this(name:String) {
         this()
         this.name = name
     }
 }
 ```
 将会生成四个方法：
 1. name: String
 2. name_ = (newValue : String):Unit
 3. getName() : String
 4. setName(newValue: String):Unit

嵌套类
在scala中，你几乎可以在任何语法结构中内嵌任何语法结构。
你可以在函数中定义函数，在类中定义类。

#### 类和对象
Scala中没有静态方法，它有一个类似的特性，叫单例对象。
通常，一个类对应有一个伴生对象，伴生对象的名字和类的名字相同，
类和伴生对象之间可以相互访问私有的方法和属性，它们必须存在于同一个源文件中。
伴生类生成的类中带一个$符号。
Scala 中伴生对象采用 object 关键字声明，伴生对象中声明的全是静态内容，可以通过伴生对象名称直接调用。
apply方法
调用object对象是默认执行的方法就是apply方法，这个方法通常用来返回伴生类的对象。
每个scala程序都必须从一个对象的main方法开始，可以使用App Trait或自己写main.
```scala
object Hello extends App {
    println("hello world")
}

object Test {
    def main(args: Array[String]) {
        println("hello world")
    }
}
```