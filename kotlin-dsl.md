# Kotlin构建自定义DSL

>作者：丁筱颜
>更好的阅读体验，点击[阅读原文]

[TOC]


## DSL（Domain Specified Language）领域专用语言

### 常见的DSL

- 正则表达式

通过一些规定好的符号和组合规则，通过正则表达式引擎来实现字符串的匹配

- HTML&CSS

虽然写的是类似XML 或者 .{} 一样的字符规则，但是最终都会被浏览器内核转变成Dom树，从而渲染到Webview上

- SQL

虽然是一些诸如 create select insert 这种单词后面跟上参数，这样的语句实现了对数据库的增删改查一系列程序工作

### DSL分类

- 内部 DSL（从一种宿主语言构建而来）
- 外部 DSL（从零开始构建的语言，需要实现语法分析器等）

## 通过Kotlin构建类型安全的DSL

例子：

```html
html {
            head {
                title { +"XML encoding with Kotlin" }
            }
            body {
                h1 { +"XML encoding with Kotlin" }
                p { +"this format can be used as an alternative markup to XML" }

                // 一个具有属性和文本内容的元素
                a(href = "http://kotlinlang.org") { +"Kotlin" }

                // 混合的内容
                p {
                    +"This is some"
                    b { +"mixed" }
                    +"text. For more see the"
                    a(href = "http://kotlinlang.org") { +"Kotlin" }
                    +"project"
                }
                p { +"some text" }

                // 以下代码生成的内容
                p {
                    for (arg in args)
                        +arg
                }
            }
        }
```

这是在kotlin中完全合法的一段代码，并且可以正确运行出结果，得到的结果如下图

```html
<html>
  <head>
    <title>
      XML encoding with Kotlin
    </title>
  </head>
  <body>
    <h1>
      XML encoding with Kotlin
    </h1>
    <p>
      this format can be used as an alternative markup to XML
    </p>
    <a href="http://kotlinlang.org">
      Kotlin
    </a>
    <p>
      This is some
      <b>
        mixed
      </b>
      text. For more see the
      <a href="http://kotlinlang.org">
        Kotlin
      </a>
      project
    </p>
    <p>
      some text
    </p>
    <p>
    </p>
  </body>
</html>

```

这就是我们自定义DSL构造器得出的结果。

首先我们回顾一些kotlin技术：

###lambda与高阶函数

Kotlin 的 lambda 有个规约：如果 lambda 表达式是函数的最后一个实参，则可以放在括号外面，并且可以省略括号，这个规约是 Kotlin DSL 实现嵌套结构的本质原因。

传递lambda表达式作为参数：```fun html(init: HTML.() -> Unit): HTML``，这个方法接收一个有receiver的lambda表达式，因为这样在block的内部就可以直接访问receiver的公共成员了，这一点也很重要。

###扩展函数（扩展属性）

对于同样作为静态语言的 Kotlin 来说，扩展函数（扩展属性）是让他拥有类似于动态语言能力的法宝，即我们可以为任意对象动态的增加函数或属性。

比如，为 LocalDate 扩展一个函数：` toDate()`:

```kotlin
fun LocalDate.toDate(): Date = Date.from(this.atStartOfDay().atZone(ZoneId.systemDefault()).toInstant())

```

与 JavaScript 这类动态语言不一样，Kotlin 实现原理是： 提供静态工具类，将接收对象(此例为 String )做为参数传递进来,以下为该扩展函数编译成 Java 的代码

```java
 @NotNull
   public static final Date toDate(@NotNull LocalDate $receiver) {
      Intrinsics.checkParameterIsNotNull($receiver, "$receiver");
      Date var10000 = Date.from($receiver.atStartOfDay().atZone(ZoneId.systemDefault()).toInstant());
      Intrinsics.checkExpressionValueIsNotNull(var10000, "Date.from(this.atStartOf…emDefault()).toInstant())");
      return var10000;
   }

//java call
Date date = toDate(LocalDate.now());
//kotlin call
val date = LocalDate.now().toDate()
```

在Kotlin语言中，类不再是语言的最小单位。我们既可以单独声明一个全局函数，也可以声明全局变量。因此，你可以认为toLong是一个函数整体，这个函数的接收者可以是任意对象。

而对于Java，Java语言中所有的行为都必须在类体中完成。具体到某个函数或某一个变量始终属于某一个类实例。换而言之，其Receiver是固定的，也就没有了所谓Receiver的概念。

### 一元运算符

```kotlin
operator fun BigDecimal.unaryPlus() = this.plus(java.math.BigDecimal.TEN)

println(+BigDecimal("100"))
//110
```

## 开始分析

### 实现原理

首先看

```
html{
    head{}
    body{}
}
```

这个代码块的实质是一个函数调用

```kotlin
fun html(init: HTML.() -> Unit): HTML {
    val html = HTML()
    html.init()
    return html
}
```

这个函数接受一个名为 `init` 的参数，该参数本身就是一个函数。 该函数的类型是 `HTML.() -> Unit`，它是一个 *带接收者的函数类型* 。 这意味着我们需要向函数传递一个 HTML 类型的实例（ *接收者* ）， 并且我们可以在函数内部调用该实例的成员。 该接收者可以通过 *this* 关键字访问：

```
html {
    this.head { …… }
    this.body { …… }
}
```

（`head` 和 `body` 是 `HTML` 的成员函数。）

现在，像往常一样，*this* 可以省略掉了，我们得到的东西看起来已经非常像一个构建器了：

```
html {
    head { …… }
    body { …… }
}
```

它创建了一个 `HTML` 的新实例，然后通过调用作为参数传入的函数来初始化它 （在我们的示例中，归结为在HTML实例上调用 `head` 和 `body`），然后返回此实例。 这正是构建器所应做的。

`HTML` 类中的 `head` 和 `body` 函数的定义与 `html` 类似。 唯一的区别是，它们将构建的实例添加到包含 `HTML` 实例的 `children` 集合中：

```kotlin
fun head(init: Head.() -> Unit) : Head {
    val head = Head()
    head.init()
    children.add(head)
    return head
}

fun body(init: Body.() -> Unit) : Body {
    val body = Body()
    body.init()
    children.add(body)
    return body
}
```

实际上这两个函数做同样的事情，所以我们可以有一个泛型版本，`initTag`：

```kotlin
protected fun <T : Element> initTag(tag: T, init: T.() -> Unit): T {
    tag.init()
    children.add(tag)
    return tag
}
```

所以，现在我们的函数很简单：

```kotlin
fun head(init: Head.() -> Unit) = initTag(Head(), init)

fun body(init: Body.() -> Unit) = initTag(Body(), init)
```

并且我们可以使用它们来构建 `<head>` 和 `<body>` 标签。

这里要讨论的另一件事是如何向标签体中添加文本。在上例中我们这样写到：

```
html {
    head {
        title {+"XML encoding with Kotlin"}
    }
    // ……
}
```

所以基本上，我们只是把一个字符串放进一个标签体内部，但在它前面有一个小的 `+`， 所以它是一个函数调用，调用一个前缀 `unaryPlus()` 操作。 该操作实际上是由一个扩展函数 `unaryPlus()` 定义的，该函数是 `TagWithText` 抽象类（`Title` 的父类）的成员：

```kotlin
operator fun String.unaryPlus() {
    children.add(TextElement(this))
}
```

所以，在这里前缀 `+` 所做的事情是把一个字符串包装到一个 `TextElement` 实例中，并将其添加到 `children`集合中， 以使其成为标签树的一个适当的部分。

### 作用域控制

由于内部的作用域默认可以获得外部的隐式接收器

```
html { 
            head { 
                head{
                    //无意义的head
                }
            }
        }
```

我们可以使用`@DslMarker`来注释一个注解

```kotlin
@Target(ANNOTATION_CLASS)
@Retention(BINARY)
@MustBeDocumented
@SinceKotlin("1.1")
public annotation class DslMarker

@DslMarker
annotation class HtmlTagMarker
```

注释类`HtmlTagMarker`被称为*一个DSL标记*，它被注解`@DslMarker`注释。

一般规则：

- 如果隐式接收*器用@HtmlTagMarker*相应的DSL标记注释标记，则它可以*属于* DSL
- 同一DSL的两个隐式接收器在同一范围内不可访问
  - 就近原则
  - 其他可用的接收器可以照常解析，但如果得到的解析调用绑定到这样的接收器，则编译错误

标记规则：隐式接收器被视为被`@HtmlTagMarker`注释，需要满足下面的条件：

- 它的类型是被标记，或
- 它的类型分类器被标记
  - 或其任何超类/超接口

补充说明

- `this@label`无论是否被标记，都可以访问接收器

### 完整代码

```kotlin
package com.example.html

interface Element {
    fun render(builder: StringBuilder, indent: String)
}

class TextElement(val text: String) : Element {
    override fun render(builder: StringBuilder, indent: String) {
        builder.append("$indent$text\n")
    }
}

@DslMarker
annotation class HtmlTagMarker

@HtmlTagMarker
abstract class Tag(val name: String) : Element {
    val children = arrayListOf<Element>()
    val attributes = hashMapOf<String, String>()

    protected fun <T : Element> initTag(tag: T, init: T.() -> Unit): T {
        tag.init()
        children.add(tag)
        return tag
    }

    override fun render(builder: StringBuilder, indent: String) {
        builder.append("$indent<$name${renderAttributes()}>\n")
        for (c in children) {
            c.render(builder, indent + "  ")
        }
        builder.append("$indent</$name>\n")
    }

    private fun renderAttributes(): String {
        val builder = StringBuilder()
        for ((attr, value) in attributes) {
            builder.append(" $attr=\"$value\"")
        }
        return builder.toString()
    }

    override fun toString(): String {
        val builder = StringBuilder()
        render(builder, "")
        return builder.toString()
    }
}

abstract class TagWithText(name: String) : Tag(name) {
    operator fun String.unaryPlus() {
        children.add(TextElement(this))
    }
}

class HTML : TagWithText("html") {
    fun head(init: Head.() -> Unit) = initTag(Head(), init)

    fun body(init: Body.() -> Unit) = initTag(Body(), init)
}

class Head : TagWithText("head") {
    fun title(init: Title.() -> Unit) = initTag(Title(), init)
}

class Title : TagWithText("title")

abstract class BodyTag(name: String) : TagWithText(name) {
    fun b(init: B.() -> Unit) = initTag(B(), init)
    fun p(init: P.() -> Unit) = initTag(P(), init)
    fun h1(init: H1.() -> Unit) = initTag(H1(), init)
    fun a(href: String, init: A.() -> Unit) {
        val a = initTag(A(), init)
        a.href = href
    }
}

class Body : BodyTag("body")
class B : BodyTag("b")
class P : BodyTag("p")
class H1 : BodyTag("h1")

class A : BodyTag("a") {
    var href: String
        get() = attributes["href"]!!
        set(value) {
            attributes["href"] = value
        }
}

fun html(init: HTML.() -> Unit): HTML {
    val html = HTML()
    html.init()
    return html
}
```



## 应用

- kotlin官方html构造器：[kotlinx.html](https://github.com/Kotlin/kotlinx.html)
- kotlin的javaFX框架：[TornadoFX](https://github.com/edvin/tornadofx)

- 安卓布局框架：[anko](https://github.com/Kotlin/anko)
- kotlin服务端框架：[ktor](https://github.com/ktorio/ktor)

## 参考文档

[类型安全的构建器](https://www.kotlincn.net/docs/reference/type-safe-builders.html)

[Scope control for implicit receivers](https://github.com/Kotlin/KEEP/blob/master/proposals/scope-control-for-implicit-receivers.md)

[Kotlin之美——DSL篇](https://www.jianshu.com/p/f5f0d38e3e44)
