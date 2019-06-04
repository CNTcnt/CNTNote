[TOC]



# Android数据库高手秘籍(九)——赶快使用LitePal 2.0版本吧

2018年06月11日 07:34:42阅读数：55

转载请注明出处：[https://blog.csdn.net/guolin_blog/article/details/80586028](https://blog.csdn.net/guolin_blog/article/details/80586028)

> 本文同步发表于我的微信公众号，扫一扫文章底部的二维码或在微信搜索 **郭霖** 即可关注，每个工作日都有文章更新。

今天很高兴告诉大家一个好消息，LitePal又出新版本了。

算了一下，上个版本1.6.1已经是半年前推出的了，而整个开源项目自2014年推出以来，我已经维护了有四年之久。这四年以来，我不断地完善着LitePal的代码，修复各种大家提出的bug，以及补充各式各样好用的新功能。而今天，时隔半年，LitePal终于迎来了一次大的版本更新，正式发布了2.0.0版本！

从1.6.1直接跨越到2.0.0，说明这次的升级变化还是非常大的。在2.0.0版本当中，我重构了内部大量的代码，使得LitePal整体的架构更加合理和清晰，API接口更加科学，并且重写了数据库的同步处理机制，解决了很多并发操作数据库的问题。最重要的是，LitePal 2.0.0版本现在全面支持Kotlin了！以后不管你是用Java还是Kotlin开发Android程序，都可以100%兼容地使用LitePal，是不是有点小激动呢？那么下面我们就来具体学习一下如何使用新版本的LitePal吧。 

# 升级到2.0.0

升级的方式很简单，如果你使用的是Android Studio，只需要在build.gradle中修改一下配置即可：

```
dependencies {
    implementation 'org.litepal.android:core:2.0.0'
}123
```

如果你使用的还是Eclipse，那么可以点击 [这里](https://github.com/LitePalFramework/LitePal/tree/master/downloads) 下载最新版的jar包。 

# 新版本变化

需要大家注意的是，2.0.0版本中几乎所有的API接口全部都变了。但是请不要惊慌，2.0.0版本是完全向下兼容的，也就是说，大家不用担心升级之后会出现大量的错误，之前所有的代码都还是可以正常运行的，只是原来旧的API会被标识为废弃，提醒大家尽快使用新的API而已，如下图所示：

![](https://img-blog.csdn.net/20180606095112168)

当然，大家也不用一听到所有的API都变了就觉得恐慌，其实一切的变更都是有规律可循的。

那么我们都知道，LitePal本身的用法就非常简单，因此升级新API的过程也同样是非常简单的，下面我们一步步来看。

首先是实体类的继承要进行修改，这是我们过去的写法：

![](https://img-blog.csdn.net/20180606100820693)

可以看到，现在DataSupport类已经被标为了废弃，虽然暂时还可以正常工作，但是不建议再继续使用了，从LitePal 2.0.0版本开始建议使用LitePalSupport类，我们将代码改成如下所示即可：

![](https://img-blog.csdn.net/20180606101801174)

将实体类的继承结构更改为LitePalSupport之后，得到的一个隐形好处就是所有的实例CRUD方法都会自动升级到2.0.0版本了，如save()方法，update()方法，delete()方法等等。因此，我们原来存储一条数据该怎么写，现在就还怎么写，比如：

```
Song song = new Song();
song.setName("hello");
song.setDuration("180");
song.save();1234
```

这样就可以存储一条数据到数据库当中了，和之前的写法没有任何变化，但是却使用了LitePal 2.0.0中的最新接口了，因为这个save()方法是来自于LitePalSupport类的。

接下来第二步需要升级的是静态CRUD方法。原来所有的静态CRUD方法都是封装在DataSupport类当中的，比如刚才我们演示过的查询数据库的中数据可以这么写：

![](https://img-blog.csdn.net/20180606095112168)

而现在，所有的静态CRUD方法都被移动到了LitePal类当中，因此我们只需要将DataSupport修改为LitePal即可，其他的用法都是完全不变的，如下所示：

![](https://img-blog.csdn.net/20180606110913714)

没错，升级过程就是这么简单。总结一下其实主要就只有两点，如果你是在继承结构中使用了DataSupport，那么就将它改为LitePalSupport，如果你是调用了DataSupport中的静态方法，那么就将它改为LitePal。

不过最后还有一件事需要注意，如果你的项目代码启用了混淆，那么混淆的配置也需要进行相应的修改才行，原来的混响配置如下所示：

```java
-keep class org.litepal.** {
    *;
}

-keep class * extends org.litepal.crud.DataSupport {
    *;
}
```

而由于DataSupport类已经被废弃了，因此这里也需要将混淆文件中的DataSupport改成LitePalSupport才行，如下所示：

```java
-keep class org.litepal.** {
    *;
}

-keep class * extends org.litepal.crud.LitePalSupport{
    *;
}
```

将以上的操作都完成之后，那么恭喜你，你的代码已经完全升级到LitePal 2.0.0版本了。

# 在Kotlin中使用LitePal

Kotlin自去年Google IO大会成为Android一级语言之后，经过了一年多的发展，如今已经正式成为Google心中的亲儿子了。未来使用Kotlin编写Android程序的人会越来越多，因此LitePal也及时跟进，全面支持了Kotlin语言。

下面我来给大家简单演示下如何在Kotlin代码中使用LitePal吧。

首先要定义一个实体类，这里我们就以Book类为例吧。比如Book类中有id、name、page这三个字段，并且继承自LitePalSupport类，如下所示：

```java
class Book(val name: String, val page: Int) : LitePalSupport() {
    val id: Long = 0
}
```

可以看到，Kotlin中定义实体类真的是非常简单。需要注意的是，如果你的实体类中需要定义id这个字段，不要把它放到构造函数当中，因为id的值是由LitePal自动赋值的，而不应该由用户来指定。因此这里我们在Book类的内部声明了一个只读类型的id。

然后需要在litepal.xml中声明一下这个实体类，这个属于常规操作了：

```java
<list>
    <mapping class="org.litepal.litepalsample.model.Book" />
</list>
```

好了！接下来我们就可以进行CRUD操作了，那么由于是首次使用Kotlin来操作LitePal，这里我会将每一个操作都分别演示一下。首先是存储操作，代码如下所示：

```java
val book = Book("第一行代码", 552)
val result = book.save()
Log.d(TAG, "save result is $result , book id is ${book.id}")
```

可以看到，这里我们先创建了一个Book的实例，并传入书名和页数，然后调用save()方法就可以将这条数据存储到数据库中了。存储结束后这里还用一条打印日志打印出了执行结果，如下所示：

```java
D/MainActivity: save result is true , book id is 1
```

可以看到，这里显示存储成功，并且book的id值变成了1，说明LitePal在存储成功之后自动给id赋值了。

接下来我们到数据库中具体查看一下吧，如下图所示：

![](https://img-blog.csdn.net/20180606162935501)

再一次验证存储操作已经成功了。

接下来我们演示一下修改操作，代码如下所示：

```java
val cv = ContentValues()
cv.put("name", "第二行代码")
cv.put("page", 570)
LitePal.update(Book::class.java, cv, 1)
```

其实基本上Kotlin上的用法大家都会觉得眼熟，因为和Java都是类似的，只是具体语法可能有些不太一样。就比如update()方法接收的第一个参数是个Class对象，在Java中我们会传入Book.class，而在Kotlin中则需要传入Book::class.java。

执行上述代码，然后再到数据库中查看一下，结果如下图所示：

![](https://img-blog.csdn.net/2018060617012056)

没错，说明我们的修改操作也顺利完成了。

下面看一下删除操作，代码如下所示：

```java
LitePal.delete(Book::class.java, 1)
```

这里我们指明要删除id为1的这条记录。当然除了按照id删除以外，你还可以按照其他任意条件去删除，比如我们想把页数大于500的书全部都删掉，那么就可以这么写：

```java
LitePal.deleteAll(Book::class.java, "page > ?", "500")
```

好，现在执行上述任意一行代码，然后到数据库中观察一下，如下图所示：

![](https://img-blog.csdn.net/20180606170946909)

没有问题，可以看到这里数据库已清空，说明我们的删除操作确实生效了。

最后，再向大家演示一下查询的操作。由于现在数据库中已没有数据可查，那么我们先向库中添加两条数据，然后再执行查询操作，代码如下所示：

```java
Book("第一行代码", 552).save()
Book("第二行代码", 570).save()

LitePal.findAll(Book::class.java).forEach {
    Log.d(TAG, "book name is ${it.name} , book page is ${it.page}")
}
```

这里调用了findAll()方法，将Book表中的所有数据都查询了出来。查询的结果是一个List集合，因此我们又用了Kotlin中的forEach循环将查询到的每条记录都打印了出来。执行结果如下所示：

```java
D/MainActivity: book name is 第一行代码 , book page is 552
D/MainActivity: book name is 第二行代码 , book page is 570
```

当然，除了调用findAll()方法之外，我们还可以使用LitePal的连缀查询来对查询条件进行任意地定制：

```java
LitePal.where("name like ?", "第_行代码")
       .order("page desc")
       .limit(5)
       .find(Book::class.java)
```

这样我们就将在Kotlin中使用LitePal进行CRUD操作全部都演示完了，是不是感觉和Java中一样的简单和方便呢？当然，除了这些新功能之外，我还修复了一些已知的bug，提升了整体框架的稳定性，如果这些正是你所需要的话，那就赶快升级吧。 

# 我没学过LitePal怎么办？

本篇文章是写给已经有LitePal基础的人看的，帮助他们快速地升级到LitePal 2.0。如果你之前并没有学过LitePal，可以参考[《第一行代码 第2版》](https://blog.csdn.net/guolin_blog/article/details/52032038)第6章中的内容，里面有非常详尽的LitePal使用讲解。

另外也可以阅读我写的专栏[《Android数据库高手秘籍》](https://blog.csdn.net/column/details/android-database-pro.html)，同样对LitePal的各种使用方法进行了详细地剖析。

