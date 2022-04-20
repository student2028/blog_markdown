---
title: Scala学习Part3
date: 2021-09-09 09:09:09
comments: true #是否可评论
toc: true #是否显示文章目录
categories: "学习" #分类
tags:   #标签
    - Scala
---
scala中的方法和函数，在scala中你可以直接使用函数，但无法直接使用方法，
方法是从属于类，函数是scala中的头等公民。
#### 集合
所有集合都扩展自Iterable特质。
集合有三大类，分别是Seq,Set,Map。
对于几乎所有集合类，scala都提供了可变和不可变版本。
每个scala集合特质都有一个带apply方法的伴生对象，这个apply方法可以用来
构建该集合中的实例。
在scala中，列表要么是Nil(空表)，要么是一个head元素加上一个tail,而tail本身又是
一个列表。
```scala
val l=9::4:;2::Nil
def sum(lst: List[Int]):Int = if (lst == NIl) 0 else lst.head + sum(lst.tail)
def sum2( lst : List[Int]) :Int = lst match {
  case Nil => 0
  case h :: t => h + sum(t)
}
val lst = (1,7,2,9)
lst.reduceLeft(_ - _ ) // 1 - 7 - 2 - 9 左结合 
lst.reduceRight(_-_) // 1 - (7 - (2-9)) 右结合 
lst.foldLeft(0)(_-_) //柯里化让scala可以根据初始值的类型推断出返回值的类型定义
```
fold可以作为循环的替代，这样做并不一定是好的，但觉得循环和改值可以被消除是一件有趣的事。
#### 模式匹配和样例类(case class)
##### case class
样例类是一种特殊的类，被优化用于模式匹配。
```scala
sealed abstract class Amount
case class Dollar(value:Double) extends Amount
case class Currency(value: Double, unit:String) extends  Amount
case object Nothing extends Amount
amt match {
  case Dollar(v) =>"$" +v
  case Currency(_,u) => "Oh noes, I got" + u
  case Nothing => ""
}
abstract class Item
case class Article(description:String, price: Double) extends Item
case class Bundle(description:String , discount: Double, items: Item*) extends Item
case bundle(_,_,Article(descr,_),_*)=>...
//用@表示法将嵌套的值绑定到变量
case Bundle(_,_,art @Article(_,_), rest @ _*)
```
当声明case class时，如下事情自动发生：
1. 构造器中每一个参数都成val，除非显式声明为var
2. 在伴生对象中提供apply,unapply方法分别用于构造对象和模式匹配。
3. 将生成toString,equals,hashCode,copy方法

密封类的所有子类必须在该类相同的文件中定义，这样在编译期间它所有的子类都是可知的，因而编译器可以检查
模式语句的完整性，这是一个好的实践方法。
##### 模拟枚举
```scala
sealed abstract class TrafficLightColor
case object Red extends TrafficLightColor
case object Yellow extends TrafficLightColor
case object Green extends TrafficLightColor
color match {
  case Red => "stop"
  case Yellow => "hurry up"
  case Green => "go"
}
```
##### 偏函数
被包在花括号内的一组case语句是一个偏函数，一个并非对所有输入值都有定义的函数。它是
PartialFunction[A,B]类的一个实例，A是参数类型，B是返回值类型。
```scala
//PartialFunction不可省略
val f:PartialFunction[Char,Int] = {case '+' => 1; case '-'=> -1} 
f('-')
f('+')
f.isDefinedAt('0')//false
f('0') //throw MatchError
"-3+4".collect(f) //Vector(-1,1)
```
#### 隐式转换和隐式参数
##### **隐式转换**
隐式转换可以丰富现有类的功能。
隐式转换函数指的是那种以implicit关键字声明的带有单个参数的函数。
这样的函数会被自动应用，将值从一个类型转成另一个类型。
```scala
val contents = new File("readme").read
class RichFile(val from:File) {
  def read = Source.fromFile(from.getPath).mkString
}
```
scala会考虑如下的隐式转换函数：
1. 位于源或目标类型的伴生对象中的隐式函数。
2. 位于当前作用域可以以单个标识符指代的隐式函数
可以将引入局部化来避免不想要的转换的发生。

##### **隐式参数**
函数或方法可以带有一个标记为implicit的参数列表，这种情况下，编译器将会检查找到缺省值，
提供给函数或方法。
```scala
case class Delimiters(left:String, right:String)
def quote(what:String) (implicit delims:Delimiters) = 
delims.left + what + delims.right
quote("bonjour le monde")
//没有传递第二个隐式参数 
implicit val quoteDelimiters = Delimiters("<",">")
```