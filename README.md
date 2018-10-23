# Kotlin tips and tricks

一篇小文，说说kotlin带来方便的一些特性。

#### IntelliJ IDEA对Kotlin的支持

同一个团队的产品，所以在kotlin一出生就有完美的IDE支持。对于Java程序员要学习Kotlin，开始部分语法会遇到难题，用IDE有两个小trick：

- 直接复制Java代码到Kotlin文件中，可以直接转变为Kotlin代码，然后学习。
- 也可以通过菜单中的工具，将整个Java类转变为Kotlin代码。

但，这样的功能用得越少，学习Kotlin也就越快。

#### var和val

老生长谈了，var是可变的，val不可变，并发安全。建议val变量在定义之初就要初始化。

#### 表达式函数体

定义一个函数，可以有更简单的形式，注意是函数体只有一个语句的情况。

```java
// 一般形式
fun max(a: Int, b: Int): Int { 
    return if (a > b) a else b 
}
// 表达式函数体
fun max(a: Int, b: Int): Int = if (a > b) a else b
```
对于上面的特性，NTELLIJ IDEA也有提示，可以通过工具转换。

####类型推导
Kotlin强类型的语言，但有些场景类型可以省略，由编译器进行推断。

```java
// 类型推断
val a = SomeClass()
// 上面的函数可以重新定义
fun max(a: Int, b: Int) = if (a > b) a else b
```

####类型智能转换
最喜欢的功能之一。还记得Java里类型转换的括号嘛，我们看看Kotlin的做法

```java
// obj不需要像Java一样强制转换，!is也是一样
if (obj is String) {
    print(obj.length)
}
// 当然，这个也可以
if (obj !is String || obj.length == 0) return
// 还有强大的when
when (x) {
    is Int -> print(x + 1)
    is String -> print(x.length + 1)
    is IntArray -> print(x.sum())
}
```
除此之外，Kotlin也支持强制类型的转换，as和as

```java
// 非安全转换，会抛出异常，如果y为null
val x: String = y as String
// 安全转换，如果y为null，直接返回null
val x: String = y as? String
```

####快速的创建数组
对比Java，会有一些方便

```java
val a: Array<Int> = arrayOf(1,2,3)
val b:Array<Int> = Array(3,{k -> k*k})
```

####import重命名
算是一个特性，我们实践中很少能够用到了。

```java
// 神奇，会优先使用import进来的
import Base as Hello

class Hello {
	fun hello() {
		println("hello world")
	}
}
```

####range区间
方便的语法糖，记住类型就好

```java
// 闭区间
val range：IntRange = 1 .. 5
// 左闭右开
val range : IntRange = 1 until 5
```

####控制流表达式
与Java不同，if .. elas / try ..catch在Kotlin都是表达式，可以出现在等号的右边。

```java
val max = if (a > b) a else b
// 还有一个相关的小特性，顺便说说
fun testIfReturn(a: Int, b: Int) {
    
    val max = if (a > b) {
        print("Max a")
        a // 代码块最后一行表达式的值就是 max 的值
    } else {
        print("Max b")
        b
    }
    // 字符串插值，以前写Java羡慕了很久Perl
    println("max = $max") 
}
```

####代理（by）
代理模式提供一种实现集成的替代方法，Kotlin原生就支持。

```java
// 代理类
interface Base {
  fun print()
}
class BaseImpl(val x: Int) : Base {
  override fun print() { print(x) }
}
 
class Derived(b: Base) : Base by b
 
fun main(args: Array<String>) {
  val b = BaseImpl(10)
  Derived(b).print()  // prints 10
}
// 代理属性
// 1. 延迟属性：第一次真正访问的时候，初始化
// 2. 观察属性：属性发生改变，通知监听者
// 3. Map中存储属性，有趣
class Example {
  var p: String by Delegate()
}
class Delegate {
  operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
    return "$thisRef, delegating '${property.name}' to me!"
  } 
  operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
    println("$value assigned to '${property.name} in $thisRef.'")
  }
}
// 读取p值
val e = Example()
println(e.p)  //Example@33a17727, delegating ‘p’ to me!
// 设置p值
e.p = "NEW"
// 打印结果：
// NEW assigned to ‘p’ in Example@33a17727.
// Lazy简单用法
val lazyValue: String by lazy {
    println("computed!")
    "Hello"
}
 
fun main(args: Array<String>) {
    println(lazyValue)
    println(lazyValue)
}
// prints：
// computed!
// Hello
// Hello

// 可观察方法简单用法，这个在业务系统中，可以有不错的尝试
class User {
    var name: String by Delegates.observable("<nomalName>") {
        prop, old, new ->
        println("$old -> $new")
    }
} 
fun main(args: Array<String>) {
    val user = User()
    user.name = "first"
    user.name = "second"
}
// 结果：
// < nomalName > -> first
// first -> second
```

####sealed class
密封类用来表示受限的类继承结构：当一个值为有限集中的类型、而不能有任何其他类型时。在某种意义上，他们是枚举类的扩展：枚举类型的值集合也是受限的，但每个枚举常量只存在一个实例，而密封类的一个子类可以有可包含状态的多个实例。 

```java
// Kotlin 1.1+，否则子类必须在sealed class内部
sealed class Expr
data class Const(val number: Double) : Expr()
data class Sum(val e1: Expr, val e2: Expr) : Expr()
object NotANumber : Expr()

// 作用1 保护代码，感受不深
// 作用2 when可以没有else了
fun eval(expr: Expr): Double = when(expr) {
    is Const -> expr.number
    is Sum -> eval(expr.e1) + eval(expr.e2)
    NotANumber -> Double.NaN
    // 不再需要 `else` 子句，因为我们已经覆盖了所有的情况
}
```

####函数扩展
无需多说，之前写Go的时候，就羡慕了一把，JVM也可以了。

```java
fun Int.
```

####高阶函数
高阶函数是将函数用作参数或返回值的函数。

```java
fun <T, R> Collection<T>.fold(
    initial: R, 
    combine: (acc: R, nextElement: T) -> R
): R {
    var accumulator: R = initial
    for (element: T in this) {
        accumulator = combine(accumulator, element)
    }
    return accumulator
}
// 具体使用示例
val items = listOf(1, 2, 3, 4, 5)

// Lambdas 表达式是花括号括起来的代码块。
items.fold(0, { 
    // 如果一个 lambda 表达式有参数，前面是参数，后跟“->”
    acc: Int, i: Int -> 
        print("acc = $acc, i = $i, ") 
    val result = acc + i
    println("result = $result")
    // lambda 表达式中的最后一个表达式是返回值：
    result
})

// lambda 表达式的参数类型是可选的，如果能够推断出来的话：
val joinedToString = items.fold("Elements:", { acc, i -> acc + " " + i })

// 函数引用也可以用于高阶函数调用：
val product = items.fold(1, Int::times)
```

####带有接收者的函数字面值
带有接受者的函数类型，就是这样：

```java
A.(B) -> C
```
我第一次看到是懵的，但给一个例子，就好多了。

```java
val sum = fun Int.(other: Int): Int = this + other
println(sum(5 + 6))  // 11
```
是不是还没有透彻，来一个完整的，很强大，可以尽情玩耍。

```java
class HTML {
    fun body() { …… }
}

fun html(init: HTML.() -> Unit): HTML {
    val html = HTML()  // 创建接收者对象
    html.init()        // 将该接收者对象传给该 lambda
    return html
}

html {       // 带接收者的 lambda 由此开始
    body()   // 调用该接收者对象的一个方法
}
```
为什么可以这样呢？

```java
html {
 // ……
}
```
html是一个函数调用，它接受一个lambda表达式为参数，如下：

```java
fun html(init: HTML.() -> Unit): HTML {
    val html = HTML()
    html.init()
    return html
}
```
这个函数接受一个名为 init 的参数，该参数本身就是一个函数。 该函数的类型是 HTML.() -> Unit，它是一个 带接收者的函数类型 。 这意味着我们需要向函数传递一个 HTML 类型的实例（ 接收者 ）， 并且我们可以在函数内部调用该实例的成员。 该接收者可以通过 this 关键字访问：

```java
html {
    this.head { …… }
    this.body { …… }
}
```
省略this

```java
html {
    head { …… }
    body { …… }
}
```
完整示例可见于Kotlin官方文档。

####函数扩展
之前使用Go开发的时候，着实羡慕了一把，不过现在Kotlin也可以了，话不多说

```java
fun <T> MutableList<T>.swap(index1: Int, index2: Int) {
    val tmp = this[index1] // “this”对应该列表
    this[index1] = this[index2]
    this[index2] = tmp
}
```
这个功能这个在平时使用得也是比较多的。其本质不是真正的修改所扩展的类，而是静态的解析的，类似静态的方法，那以下的case就要注意了。

```java
open class C

class D: C()

fun C.foo() = "c"

fun D.foo() = "d"

fun printFoo(c: C) {
    println(c.foo())
}
printFoo(D())  // 这里是c
```
再来一个知乎的例子，更全面

```java
operator inline fun Int.rem(blk: () -> Unit) {
  if (Random (System.currentTimeMillis()).nextInt(100) < this) blk()
}
// 开始调用
20 % { print ("你有20%的概率看到这条信息") }
```

####中缀表示法

infix关键字，标识中缀函数：

- 它们必须是成员函数或扩展函数；
- 它们必须只有一个参数；
- 其参数不得接受可变数量的参数且不能有默认值。

```java
infix fun Int.shl(x: Int): Int { …… }

// 用中缀表示法调用该函数
1 shl 2

// 等同于这样
1.shl(2)
```

####可变数量的参数（Varargs）
函数的参数（通常是最后一个）可以用 vararg 修饰符标记，这里也可以用伸展（spread）操作符（在数组前面加 *）：

```java
fun <T> asList(vararg ts: T): List<T> {
    val result = ArrayList<T>()
    for (t in ts) // ts is an Array
        result.add(t)
    return result
}
// 如果要传一个数组
val a = arrayOf(1, 2, 3)
val list = asList(-1, 0, *a, 4)
```

####use函数
实现了Closeable接口的对象可调用use函数，自动close。不举例了。

####with、let、also、run and apply
用一张图和一段代码说明：

![](1_pLNnrvgvmG6Mdi0Yw3mdPQ.png)

代码如下：

```java
// 代码转载自简书
class Resp<T> {
    var code: Int = 0
    var body: T? = null
    var errorMessage: String? = null

    fun isSuccess(): Boolean = code == 200

    override fun toString(): String {
        return "Resp(code=$code, body=$body, errorMessage=$errorMessage)"
    }
}
fun main(args: Array<String>) {
    var resp: Resp<String>? = Resp()
    if (resp != null) {
        if (resp.isSuccess()) {
            // do success
            println(resp.body)
        } else {
            println(resp.errorMessage)
        }
    }

    resp?.run {
        if (isSuccess()) {
            // do success
            println(resp.body)
        } else {
            println(resp.errorMessage)
        }
    }

    resp?.apply {
        if (isSuccess()) {
            // do success
            println(resp.body)
        } else {
            println(resp.errorMessage)
        }
    }

    resp?.let {
        if (it.isSuccess()) {
            // do success
            println(it.body)
        } else {
            println(it.errorMessage)
        }
    }

    resp?.also {
        if (it.isSuccess()) {
            // do success
            println(it.body)
        } else {
            println(it.errorMessage)
        }
    }
}
```

####typealias

```java
typealias Cache = HasmMap<String, Int>
```


####Java调用Kotlin属性

```java
// kotlin
var a: Int = 1

// java
someone.setA(2)
someone.getA() // 2
```

####Java调用Kotlin方法

```java
// Counter.kt
fun count() = 42

// Java class
int size = CounterKt.count()
```
CounterKt？不优雅

```java
// Counter.kt
@file:JvmName("Counter")
fun count() = 42

// Java class
int size = Counter.count()
```

####Java调用Kotlin静态方法

```java
// Kotlin
companion object {
	fun count() = 42
}

// Java class
int size = Counter.Companion.count()
```
不够优雅~

```java
// Kotlin
companion object {
	@JvmStatic
	fun count() = 42
}

// Java class
int size = Counter.count()
```
####Java调用Kotlin构造函数

```java
// Kotlin
class Person(val fullName: String, val nickName: String? = null)
// Java
Person person = new Person("Lorenzo");  // error

// Kotlin
class Person @JvmOverloads constructor (
	val fullName: String,
	val nickName: String? = null
)
// Java
Person person = new Person("Lorenzo");  // ok
```



