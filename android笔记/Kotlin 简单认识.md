[TOC]

# Kotlin 简单认识

* Kotlin 中没有 new 关键字

* 无分号

  ​

## kotlin从语法上解决NullPointerException

* 在 Kotlin 中，其类型系统严格区分一个引用可以容纳 null （可空引用）还是不能容纳（非空引用）。**也就是说，一个变量是否可空必须显示声明，对于可空变量，在访问其成员时必须做空处理，否则无法编译通过。**

  ~~~kotlin
  var a: String = "abc"//默认不为空，空安全
  a = null // 编译错误
  //此时，不管你怎么操作 a 对象，都不会导致 nullPointerException
  var length = a.length

  var b:String? = "cde"//在类型后接上？表示该对象可为空
  b = null // 可以编译通过
  //此时，你操作 b 对象有可能导致 nullPointer
  var length = b.length //此时b 有可能为空，不安全，那么怎么优雅地解决这个问题呢？

  //kotlin 提供了安全调用操作符，写作 ?.
  var length:Int? = b?.length//如果 b 非空，就返回 b.length；否则返回 null。注意，这个表达式的类型是 Int?，表示可能为空。
  ~~~

  空判断

  Kotlin的空安全设计对于声明可为空的参数，在使用时要进行空判断处理，有两种处理方式，字段后加`!!`像Java一样抛出空异常，另一种字段后加`?`可不做处理返回值为 null或配合`?:`做空判断处理

  ```kotlin
  //类型后面加?表示可为空
  var age: String? = "23" 
  //抛出空指针异常
  val ages = age!!.toInt()
  //不做处理返回 null
  val ages1 = age?.toInt()
  //age为空返回-1
  val ages2 = age?.toInt() ?: -1
  ```

* 安全的类型转换

  ~~~kotlin
  //我们都知道，当我们对一个对象进行强转时，若对象不是目标类型，那么就会导致 ClassCastException。此时，我们可以使用安全的类型转换，如果尝试转换不成功则返回 null：

  val aInt: Int? = a as? Int
  //等号左侧声明一个可为空的 Int 常量接收右侧的值，而右侧表示：如果 a 是 Int 类型，则返回 a，否则返回 null。
  ~~~

* 可空类型集合

  ~~~kotlin
  //如果你有一个可空类型元素的集合，并且想要过滤非空元素，你可以使用 filterNotNull 来实现。

  val nullableList: List<Int?> = listOf(1, 2, null, 4)//List<Int?>表示列表中的对象是可以为空的
  val intList: List<Int> = nullableList.filterNotNull()//调用List方法过滤掉空对象
  ~~~

* 非空属性必须在定义的时候初始化,kotlin提供了一种可以延迟初始化的方案,使用 lateinit 关键字描述属性：

  ~~~kotlin
  public class MyTest {
      lateinit var subject: TestSubject //有 lateinit 关键字才可以先不初始化

      @SetUp fun setup() {
          subject = TestSubject()
      }

      @Test fun test() {
          subject.method()  // dereference directly
      }
  }
  ~~~

  ​

 ## 常变量

* 在Kotlin中常量用`val`声明，变量用`var`声明，关键字在前面，类型以冒号`:`隔开在后面，也可以省略直接赋值（自动匹配），类型后带问号`?`表示可为空类型(默认空安全)。

  ~~~kotlin
  //常量数组int[][][] arrs = new int[3][2][1];
  val arrs = Array(3) { Array(2) { IntArray(1) } }//常量
  //空安全变量
  var str: String = "hello"
  var str1 = "word"
  //可为空字符串变量
  var str2: String? = null
  ~~~

## 条件

* `if...else` 正常使用，不过移除了`switch`用更强大的`when`替代，when子式可以是常量、变量、返回数值的表达式、返回Boolean值的表达式，强大到用来替换`if...else if`，最好用的 when

* ~~~kotlin
  // 测试值 x = 0, -1, 1, 2, 3, 6, 10
  var x = 10
  when (x) {
      //常量
      2 -> println("等于2")
      //数值表达式
      if (x > 0) 1 else -1 -> println("大于0并等于1，或小于0并等于-1")
      //Boolean类型表达式
      in 1..5 -> println("范围匹配1-5")
      !in 6..9 -> println("不是6-9")
      is Int -> println("类型判断")
      else -> println("else")
  }
  // 代替if...else if
  when{
      x > 6 && x <= 10  ->  println("大于6小于等于10")
      x < 6 -> println("小于6")
      else -> println("else")
  }
  ~~~

## 循环

* `while` 和 `do...while` 同Java并无区别，`for`则有很大改变并多出了几个变种

  ```kotlin
  val list = arrayListOf("aa", "bb", "cc")
  //递增for (int i = 0; i < list.size(); i++)
  for (i in list.indices) {
     print(list[i])
  }
  //递增for (int i = 2; i < list.size(); i++)
  for (i in 2..list.size-1) {
     print(list[i])
  }
  //递减for (int i = list.size() - 1; i >= 0; i--)
  for (i in list.size - 1 downTo 0) {
      print(list[i])
  }
  //操作列表内的对象
  for (item in list) {
      print(item)
  }
  //加强版
  for（（i， item） in list.witnIndex()） {
      print(list[i])
      print(item)
  }
  //变种版
  list.forEach {
      print(it)
  }

  list.forEach {
      print(it)
  }

  list.forEachIndexed { i, s ->
      print(list[i])
      print(s)
  }

  list.forEachIndexed(object :(Int,String) -> Unit{
      override fun invoke(i: Int, s: String) {
          print(list[i])
          print(s)
      }
  })
  ```

## 类

* 类的修饰符包括 classModifier 和_accessModifier_:

- classModifier: 类属性修饰符，标示类本身特性。

  ```kotlin
  abstract    // 抽象类  
  final       // 类不可继承，默认属性
  enum        // 枚举类
  open        // 类可继承，类默认是final的
  annotation  // 注解类
  ```

- accessModifier: 访问权限修饰符

  ```kotlin
  private    // 仅在同一个文件中可见
  protected  // 同一个文件中或子类可见
  public     // 所有调用的地方都可见
  internal   // 同一个模块中可见
  ```



* 内部类和参数默认为public，而在Java中为 default  即在同一个包中可见

  内部类和参数默认为public，而在Java中为 default  即在同一个包中可见
  类默认为不可继承(final)，想要可被继承要声明为`open`或`abstract`

  内部类和参数默认为public，而在Java中为 default  即在同一个包中可见
  类默认为不可继承(final)，想要可被继承要声明为`open`或`abstract`
  取消了static关键字，静态方法和参数统一写在`companion object`块

  内部类和参数默认为public，而在Java中为 default  即在同一个包中可见
  类默认为不可继承(final)，想要可被继承要声明为`open`或`abstract`
  取消了static关键字，静态方法和参数统一写在`companion object`块
  internal模块内可见，inner内部类

  ​

1. 构造器：Koltin 中的类可以有一个 主构造器，以及一个或多个次构造器，主构造器是类头部的一部分，位于类名称之后:

   ```kotlin
   //主构造器中不能包含任何代码，初始化代码可以放在初始化代码段中，初始化代码段使用 init 关键字作为前缀。
   //注意：主构造器的参数可以在初始化代码段中使用，也可以在类主体定义的属性初始化代码中使用。 一种简洁语法，可以通过主构造器来定义属性并初始化属性值（可以是var或val）
   //如果构造器有注解，或者有可见度修饰符，这时constructor关键字是必须的，注解和修饰符要放在它之前。
   class Person constructor(firstName: String) {
     init {
           println("FirstName is $firstName")
       }
   }
   ```

   如果主构造器没有任何注解，也没有任何可见度修饰符，那么constructor关键字可以省略。

   ```kotlin
   class Person(firstName: String) {
   }
   //如果一个非抽象类没有声明构造函数(主构造函数或次构造函数)，它会产生一个没有参数的构造函数。构造函数是 public 。如果你不想你的类有公共的构造函数，你就得声明一个空的主构造函数：
   class DontCreateMe private constructor () {
   }
   ```

2. 次构造器

   ~~~kotlin
   //
   class Person(val name: String) {
       constructor (name: String, age:Int) : this(name) {
           // 初始化...
       }
   }
   ~~~

   ​

3. 定义属性

   ```kotlin
   var <propertyName>[: <PropertyType>] [= <property_initializer>]
       [<getter>]
       [<setter>]
   //getter 和 setter 都是可选

   //如果属性类型可以从初始化语句或者类的成员函数中推断出来，那就可以省去类型，val不允许设置setter函数，因为它是只读的。
   ```

4. 一个简单的 class

   ~~~kotlin
   class Person {

       var lastName: String = "zhang"
           get() = field.toUpperCase()   // 将变量赋值后转换为大写
           set

       var no: Int = 100
           get() = field                // 后端变量
           set(value) {
               if (value < 10) {       // 如果传入的值小于 10 返回该值
                   field = value		//这里不能写 no = value，因为会造成死循环
               } else {
                   field = -1         // 如果传入的值大于等于 10 返回 -1
               }
           }

       var heiht: Float = 145.4f
           private set
   }
   ~~~

5. 看到这里肯定对 field 充满疑问，它到底是什么？field 关键词只能用于属性的访问器

   ~~~kotlin
   这个问题对 Java 开发者来说十分难以理解，网上有很多人讨论这个问题，但大多数都是互相抄，说不出个所以然来，要说还是老外对这个问题的理解比较透彻，可以参考这个帖子：https://stackoverflow.com/questions/43220140/whats-kotlin-backing-field-for/43220314

   其中最关键的一句：Remember in kotlin whenever you write foo.bar = value it will be translated into a setter call instead of a PUTFIELD.

   也就是说，在 Kotlin 中，任何时候当你写出“一个变量后边加等于号”这种形式的时候，比如我们定义 var no: Int 变量，当你写出 no = ... 这种形式的时候，这个等于号都会被编译器翻译成调用 setter 方法；而同样，在任何位置引用变量时，只要出现 no 变量的地方都会被编译器翻译成 getter 方法。那么问题就来了，当你在 setter 方法内部写出 no = ... 时，相当于在 setter 方法中调用 setter 方法，形成递归，进而形成死循环

   ~~~

## 抽象类

* 抽象是面向对象编程的特征之一，类本身，或类中的部分成员，都可以声明为abstract的。抽象成员在类中不存在具体的实现。


* 注意：无需对抽象类或抽象成员标注open注解。

```kotlin
open class Base {
    open fun f() {}
}

abstract class Derived : Base() {
    override abstract fun f()
}
```

------

## 嵌套类

我们可以把类嵌套在其他类中，看以下实例：

```kotlin
class Outer {                  // 外部类
    private val bar: Int = 1
    class Nested {             // 嵌套类
        fun foo() = 2
    }
}

fun main(args: Array<String>) {
    val demo = Outer.Nested().foo() // 调用格式：外部类.嵌套类.嵌套类方法/属性
    println(demo)    // == 2
}
```

------

## 内部类

* 内部类使用 inner 关键字来表示。


* 内部类会带有一个对外部类的对象的引用，所以内部类可以访问外部类成员属性和成员函数。

```kotlin
class Outer {
    private val bar: Int = 1
    var v = "成员属性"
    /**嵌套内部类**/
    inner class Inner {
        fun foo() = bar  // 访问外部类成员
        fun innerTest() {
            var o = this@Outer //获取外部类的成员变量
            println("内部类可以引用外部类的成员，例如：" + o.v)
        }
    }
}

fun main(args: Array<String>) {
    val demo = Outer().Inner().foo()
    println(demo) //   1
    val demo2 = Outer().Inner().innerTest()   
    println(demo2)   // 内部类可以引用外部类的成员，例如：成员属性
}
```

* 为了消除歧义，要访问来自外部作用域的 this，我们使用this@label，其中 @label 是一个 代指 this 来源的标签。

------

## 匿名内部类

使用对象表达式来创建匿名内部类：

```kotlin
class Test {
    var v = "成员属性"

    fun setInterFace(test: TestInterFace) {
        test.test()
    }
}

/**
 * 定义接口
 */
interface TestInterFace {
    fun test()
}

fun main(args: Array<String>) {
    var test = Test()

    /**
     * 采用对象表达式来创建接口对象，即匿名内部类的实例。
     */
    test.setInterFace(object : TestInterFace {
        override fun test() {
            println("对象表达式创建匿名内部类的实例")
        }
    })
}
```



## 注意默认事项

1. 内部类和参数默认为public，而在Java中为 default  即在同一个包中可见
2. 内部类和参数默认为public，而在Java中为 default  即在同一个包中可见
3. 类默认为不可继承(final)，想要可被继承要声明为`open`或`abstract`
4. 内部类和参数默认为public，而在Java中为 default  即在同一个包中可见
5. 类默认为不可继承(final)，想要可被继承要声明为`open`或`abstract`
6. 取消了static关键字，静态方法和参数统一写在`companion object`块
7. 内部类和参数默认为public，而在Java中为 default  即在同一个包中可见
8. 类默认为不可继承(final)，想要可被继承要声明为`open`或`abstract`
9. 取消了static关键字，静态方法和参数统一写在`companion object`块
10. internal模块内可见，inner内部类

## 冒号：

* 在Kotlin中冒号`:`用万能来称呼绝不为过。常量变量的类型声明，函数的返回值，类的继承都需要它

  ```kotlin
  //val表示常量 var表示变量声明
  val name: String = "tutu" 

  //省略类型说明
  var age = "23"

  //fun表示函数
  fun getName(): String{
     return "tutu"
  }

  //类继承
  class UserList<E>(): ArrayList<E>() {
      //...
  }

  ```

  除此之外还有一个特别的地方也需要它，使用Java类的时候。Kotlin最终会还是编译成Java字节码，使用到Java类是必然的，在Kotlin语法如下

  ```kotlin
  val intent = Intent(this, MainActivity::class.java)
  ```

## @符

* 除了冒号另一个重要符号`@`，想必用到内部类和匿名内部类的地方一定很多，再加上支持lambda语法，没有它谁告诉你`this`和`return`指的是哪一个

  ~~~kotlin
  class User {
      inner class State{
          fun getUser(): User{
              //返回User
              return this@User
          }
          fun getState(): State{
              //返回State
              return this@State
          }
      }
  }
  ~~~

## 字符串模板

* 只需用`$`后面加上参数名，复杂的参数要加上`{}`


* ~~~kotlin
  val user = User()
  //赋值
  user.name = "tutu"
  user.age = "23"
  //取值
  val name = user.name
  val age = user.age
  var userInfo = "name:${user.name},  age:$age"
  //输出结果：name:tutu, age:23
  ~~~

## kotlin和Java

1. Kotlin特色

* Java的`getter/setter`方法自动转换成属性，对应到Kotlin属性的调用

```kotlin
public class User {
    private String name;
    private String age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAge() {
        return age;
    }

    public void setAge(String age) {
        this.age = age;
    }
}

```

* 反之Kotlin的属性自动生成Java的`getter/setter`方法，方便在Java中调用，同样的定义在Kotlin中

```kotlin
class User {
    var name: String? = null
    var age: String? = null
}

```

这样一个Java类在Kotlin中只需这样调用

```kotlin
val user = User()
//赋值
user.name = "tutu"
user.age = "23"
//取值
val name = user.name
val age = user.age

```

我们的`getter/setter`方法有时不会这么简单，这就需要自定义`getter/setter`了，另起一行设置get()/set(value)方法，实现一个Java中常用的单例，这里只为了展示，单例在Kotlin有更简单的方法实现，只要在 package 级别创建一个 object 即可

```kotlin
class User {
    companion object {
        @Volatile var instance: User? = null
            get() {
                if (field == null) {
                    synchronized(User::class.java) {
                        if (field == null)
                            field = User()
                    }
                }
                return field
            }
    }

    var name: String? = null
    var age: String? = null
}

```

自定义`getter/setter`重点在`field`，跟我们熟悉所Java的`this`指代当前类一样，`field`指代当前参数，直接使用参数名`instance`代替不会报错但单例就没效果了





