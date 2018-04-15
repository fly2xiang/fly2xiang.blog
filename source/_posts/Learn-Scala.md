title: "Scala 语法入门"
date: 2018-04-15 19:20:17
tags:
- Hadoop
- 大数据
- Spark
- Scala
categories: 
- 大数据

## 概述

在使用 Spark 时，很多的例子是使用 Scala 语言进行编写，因此有必要对 Scala 的语法与使用进行学习。

Scala 是面向对象、函数式、静态类型编程语言。

Scala 提供了命令行，可以以命令行方式执行。

Scala 可以运行于 JVM 环境，能够与 Java 语言很方便地交互，同样能够访问很多的 Java 库。

## 简单例子

### 表达式

```scala
1 + 1
println(1)
println("Hello," + "World")
```

### 值

使用 `val` 定义 （类似于常量）

```scala
val a = 1 + 1	//类型推断
a = 2	//编译错误，不能重新赋值给 "值"
val b: Int = 1 + 1   //指定类型
```

### 变量

使用 `var` 定义

```scala
var a = 1 + 1
a = 3	//变量可以重新赋值
var b: String = "foo"
```

### 块

以 `{ }` 包裹，注意块的值

···scala
println({
	val a = 1 + 1
	a + 1
})	// 输出 3
```

### 函数

支持 Lambda 方式的函数定义

```scala
(a: Int) => a + 1

val addOne = (a: Int) => a + 1
println(addOne(1))	// 输出 2

val foo = () => 8
println(foo())	// 输出 8
```

### 方法

方法与函数非常类似，但有几点关键的不同

* 方法使用 `def` 定义，`def` 之后跟上方法的名称，然后是参数列表，然后是返回值类型，最后是方法的内容

```scala
def addOne(a: Int): Int = a + 1
println(addOne(1))	// 输出 2
```

* 方法可以拥有多个参数列表，或者无参数列表

```scala
def addTwo(a: Int)(b: Int): Int = a + b
println(addTwo(1)(2)) // 输出 3

def foo: String = "bar"
println(foo) 	// 输出 bar

```

方法和函数在内容较多时可以使用块，但 Scala 没有 `return` 语句，返回值即为块的值

```scala
val foo = () => {
    val a = 1
	val b = 2
	a + b
}
println(foo())

def foo: Int = {
    val a = 1;
	val b = 2;
	a + b
}
println(foo)
```

### 类

```scala
class Foo(a: Int, b: Int) {
    def show(): Unit = {
	    println("a = " + a + ", b = " + b)
	}
}

val foo = new Foo(1, 2)
foo.show()
```

这里返回类型 `Unit` 与其他语言的 `void` 类似

### case class

case class 默认是不可变 (immutable) 的，并以值来比较

```scala
case class Rect(w: Int, h: Int)
val rect = Rect(3, 4)
val rect2 = Rect(3, 4)

if (rect == rect2) {
	println("they are same")	//输出
}
```

实例化 case class 不需要使用 `new` 关键字，直接通过 `==` 比较

### 对象

对象类似于单实例的类，可以理解为静态类

```scala
object IdGenerator {
    private var count = 0
	def create(): Int = {
	    count += 1
		count
	}
}

val id = IdGenerator.create()
println(id)	// 输出 1
val id2 = IdGenerator.create()
println(id2) // 输出 2
```

### Traits

Traits 可以包含字段与方法，多个 Traits 可以组合

```scala
trait Foo {
    def bar(a: Int): Int
}

trait Foo2 {
    def bar(a: Int): Int = {
	    a * a
	}
}

class Foo3 extends Foo2

class Foo4(b: Int) extends Foo2 {
    override def bar(a: Int): Int = {
	    (a * a) + b
	}
}

val f = new Foo4(2)
println(f.bar(2)) // 输出 6
```

### 主方法

程序的入口

```
object Main {
    def main(args: Array[String]): Unit {
	    println("Welcome")
	}
}
```

## 类型

所有类型都继承自 `Any`，`Any` 有 `equals`、`hashCode`、`toString` 方法。 
`Any` 有两个子类，分别是 `AnyVal` 和 `AnyRef` 。
`AnyVal` 是值类型的祖先，不能是 `null`， 有九个预定义的类型： `Double`、`Float`、`Long`、`Int`、`Short`、`Byte`、`Char`、`Unit`、`Boolean`。
`AnyRef` 是引用类型的祖先，如果 Scala 运行与 JVM 环境， `AnyRef` 与 `java.lang.Object` 是一样的。

```scala
val list: List[Any] = List(
  "a string",
  732,  // an integer
  'c',  // a character
  true, // a boolean value
  () => "an anonymous function returning a string"
)

list.foreach(element => println(element))
```

`Nothing` 类型是所有类型的子类型，`Nothing` 没有值
`Null` 类型是所有引用类型的子类

### 类型转换

值类型可以从低范围的类型转换到高范围的类型 

```
Byte --> Short --> Int --> Long --> Float --> Double
                    ↑
                    |
                   Char
```

## 类

### 定义类

```scala
class Foo
val foo = new Foo

class Rect(var w: Int = 1, var h: Int = 1) {
    def zoom(x: Int, y: Int): Unit = {
        w = w + x
        h = h + y
    }
	
	override def toString(): String = {
	    s"w = $w, h = $h"
	}
}

val rect = new Rect(3, 4)
rect.zoom(1, 2)
println(rect)
```

### Getter/Setter 语法

```scala
class Rect {
    private var _w = 0
	private var _h = 0
	
	def w = _w
	def h = _h
	def w_= (value: Int): Unit = {
	    _x = value
	}
	
	def h_= (value: Int): Unit = {
	    _h = value
	}
}
val rect = new Rect
rect.w = 3
rect.h = 4
```

在构造器中的参数，如果使用 `val` 或 `var` 修饰则代表他们是 `public` 的，否则则代表是 `private` 的。使用 `val` 修饰时不可被修改。


## 类型组合与混合

```scala
abstract class A {
  val message: String
}
class B extends A {
  val message = "I'm an instance of class B"
}
trait C extends A {
  def loudMessage = message.toUpperCase()
}
class D extends B with C

val d = new D
println(d.message)  // I'm an instance of class B
println(d.loudMessage)  // I'M AN INSTANCE OF CLASS B
```

```scala
abstract class AbsIterator {
  type T
  def hasNext: Boolean
  def next(): T
}

class StringIterator(s: String) extends AbsIterator {
  type T = Char
  private var i = 0
  def hasNext = i < s.length
  def next() = {
    val ch = s charAt i
    i += 1
    ch
  }
}

trait RichIterator extends AbsIterator {
  def foreach(f: T => Unit): Unit = while (hasNext) f(next())
}

object StringIteratorTest extends App {
  class RichStringIter extends StringIterator("Scala") with RichIterator
  val richStringIter = new RichStringIter
  richStringIter foreach println
}
```

## 高阶函数

函数是函数式编程语言的一等公民，在 Scala 中，将使用函数作为参数或返回函数的方法和函数称为 “高阶函数”。

```scala
val seq = Seq(1,2,3)
val seq2 = seq.map(x => x * 2)
val seq2 = seq.map(_ * 2)
println(seq2) // List(2, 4, 6)
```

Scala 支持嵌套的函数与方法定义。

## 正则表达式

```
val regex = "[0-9]".r
regex.findFirstMatchIn("foobar1") match {
    case Some(_) => println("Match a Number")
	case None => println("No Number")
}
```

Scala 使用 `"""` 来使用多行字符串，注意第二行开始的 `|` ，最后使用 `stripMargin` 会将行首的 `|` 与其之前的空白去掉。

```scala

var name =
  """line0
    |line1
    |line2
    |line3
  """.stripMargin
println(name)
```








