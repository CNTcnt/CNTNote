### 创建和升级数据库

[TOC]



#### 简介

SQLite是一款轻量级的关系型数据库，不仅支持标准的SQL语法，还遵循了数据库的ACID事务；

适合存储数据量大，结构性复杂的数据；

***

#### 创建数据库

1. 借助于**SQLiteOpenHelper**帮助类，这是一个**抽象类**，使用时用创建类 来继承它，并必须重写2个方法。
2. 一个是**onCreate(SQLiteDatabase db)**,它是在数据库创建时（仅仅打开数据库则不调用）调用的方法;db.execSQL(表的字符串承接如：CREATE_BOOK);
3. 一个是**onUpgrade(SQLiteDatabase db,int oldVersion,int newVersion)**,用于对数据库进行升级，当构造方法中的版本参数增大时，这个方法则会被调用，否则不调用。在重新创建库时，如果库中已经存在要更新的表，则要先把表删除掉，db.execSQL("**drop table if exists** Book")在调用onCreate()方法重新创建，否则直接报错。
4. SQLiteOpenHelper有2个构造方法，一般用参数少的那个，有4个参数，第一个为Context，第二个为数据库名，创建数据库时就使用这个名称，第3个参数允许我们在查询数据的时候返回一个自定义的Cursor，一般传入null，第4个为当前数据库的版本号，可用于对数据库进行升级操作；
5. 创建数据库借助于SQLiteOpenHelper**的对象**来调用它的方法，有2个方法可创建数据库，一个是**getReadableDatabase**（），一个是**getWritableDatabase**（）方法。都可创建或打开一个现有的数据库，并返回一个可对数据库进行读写操作的对象。不同的是，当数据库不可写入的时候（如磁盘空间以满），getReadableDatabase（）返回的对象将以只读的方式去打开数据库，而getWritableDatabase（）方法出现异常；
6. 数据库的信息则由SQLiteOpenHelper对象来决定；

***

#### 创建一张Book表

例：用字符串承接：public static final String CREATE_BOOK = “**create table **book(

​                      									"+"id integer primary key autoincrement,**//**integer是整形

​	**//**autoincrement表示id列是自增长			"+"name   text,**//**text是文本类型

​	**//**primary key表示将id列设为主键			 "+"author   text,

​											"+"price    real)”**//**real是浮点型

***

### 使用SQLiteDatabase操作数据

* 对数据操作无非4种，即CRUD，C为添加（Create），R为查询（Retrieve），U为更新（Update），D为删除（Delete）；通过调用SQLiteOpenHelper的getReadableDatabase（）或getWritableDatabase（）返回的SQLiteDatabase对象，可借助这个对象对数据进行操作。

#### 添加数据

* SQLiteDatabase中提供了一个用于添加数据的方法：insert（表名，null,ContentValues contentValues）;

  第一个参数是表名，希望向哪张表中传入数据，这里就传入该表的名字；

  第二个参数是用于在未指定添加数据的情况下给某些可为空的列自动赋值NULL，一般不用这个功能，直接传入null即可；

  第三个参数是一个ContentValues对象，它提供了一系列的put（）方法重载，用于向ContentValues中添加数据，只需要将表中的每个列名以及相应的待添加数据传入即可。这个对象用于储存数据；

  如contentValues.put("id",2);//用put方法把要传进Book表的id项的数据2传入。

#### 删除数据

* SQLiteDatabase中提供了一个用于删除数据的方法：delete（表名，第二个参数，第三个参数），第二个和第三个参数是用于约束删除某一行或满足删除条件的即可删除，如不指定就是默认删除8所有行。

  如：sQLiteDatabase.delete("Book","pages>?","new String[] {"500"}");//删除在book表中pages大于500的行；

#### 查询数据

* SQLiteDatabase提供了一个用于查询数据的方法：query（），这个方法有至少7个参数；调用query（）方法后返回一个Cursor对象，查询到的所有数据都将从这个对象中取出。

  第一个参数：指定查询的表名；

  第二个参数：指定查询的列名，若不指定，则默认查询所有的列；

  第三个参数：指定where的约束条件；

  第四个参数：为where的占位符提供具体的值；

  第五个参数：指定需要group by的列；

  第六个参数：对group by后的结果进一步约束；

  第七个参数：指定查询结果的排列方式；

  > ```java
  > Cursor cursor = db.query("Book",null,null,null,null,null,null);
  > if(cursor.moveToFirst())//将数据指针移动到第一行的位置
  > {
  >        do{
  >             int id = cursor.getInt(cursor.getColumnIndex("id"));//得到id列索引
  >             String author = cursor.getString(cursor.getColumnIndex("author"));
  >             int pages = cursor.getInt(cursor.getColumnIndex("pages"));
  >          }while(cursor.moveToNext());
  >    }
  > ```




***

### 使用LitePal操作数据

* 简介：LitePal是一款开源的Android数据库框架，采用了对象关系映射（ORM）的模式，将平时开发时最常用到的一些数据库功能进行了封装。
* ORM:我们使用的编程语言是面向对象语言，而使用的数据库是关系型数据库，那么将面向对象的语言和面向关系的数据库之间建立一种映射关系，这就是对象关系映射了，即ORM。它赋予一个强大的功能，即可以用面向对象的思维来操作数据库，而不用再和SQL语句打交道。
* 配置LitePal，大多数开源项目都会将版本号提交到jcenter上，所以只需在app/build.gradle文件中声明该开源库的引用即可：compile ‘org.litepal.android:core:1.4.1’
* 接下来配置litepal.xml文件，在app/src/main目录new一个assets目录，然后在这个目录下新建一个litepal.xml文件


#### 创建和升级数据库

* 创建一个类(称为模型类)，如Book类，Book类就会对应数据库中的Book表，而Book类中的数据成员如author即对应表中的列，这就是对象关系映射的最直观体验。创建完类后，还需将Book表添加到映射模型列表当中，此时修改刚刚创建的litepal.mxl文件。

>```java
><?xml version="1.0" encoding="utf-8" ?>
>    <litepal>
>    <dbname value="BookStore"></dbname>
>    <version value="1"></version>
>    <list>
>        <mapping class="com.cnt.litepaltest.Book"></mapping>
>    </list>
></litepal>
>```

* dbname 指定数据库名，version指定版本，mapping（映像）来声明要配置的映射模型类。
* 创建数据库:LitePal.getDatabase();
* 升级数据库即要改变数据的表或添加表时，直接修改或添加，然后在litepal.mxl中修改版本号就行。

#### 使用LitePal添加数据

* 进行表管理操作时LitePal不需要模型类有任何继承结构，**但是进行CRUD操作时就需要让模型类继承DataSupport**;


* 先让模型类继承DataSupport，然后就可以在活动中直接创建模型类的实例，将要储存的数据设置好，然后调用一下sava（）方法即可。

#### 使用LitePal更新数据

* 已储存的对象：对于LitePal来说，对象是否已存储是根据调用（这里model表示模型类的实例）model.isSaved（）来判断，如果返回true，则表示已存储，如果返回false则表示未存储。只有2中情况返回true，一种是已经调用过model.save()方法添加过对象了；第二种是model对象是通过LitePal提供的查询API查出来的，由于是从数据库查到的对象，因此也会被认为是已存储的对象。

1. 对已储存的对象进行重新设值：然后重新调用save方法即可。

2. new一个模拟类对象，然后把要改变的设置进去，然后在调用upDataAll（添加约束条件）（如不添加，则更新所有的指定的列）方法来更新。

   >```java
   > Book book = new Book();
   > book.setPrice(14.55);
   > book.setPress("Anchor");
   > book.upDataAll("name = ?and author = ?","The Lost Symnol","DanBrown");
   >```

   意思的将name为The Lost Symnol，author为DanBrown的数据行的Price更新为14.55，Press更新为Anchor。

3. 特别：当想把一个字段的值更新为默认值的花，不可用上面的方法来set数据，应该调用setToDefault（添加要更新的列）方法。

   >```java
   >Book book = new Book();
   > book.setToDefault("pages");
   > book.upDataAll();
   >```

   意思是将全部数据行的pages设为默认值。

#### 使用LitePal删除数据

* 有2种删除数据的方式，第一种直接调用已储存对象的delete（）方法即可；第二种是用DataSupport.deleteAll（指定删除哪张表，添加约束条件），不添加则删除表中的全部数据。

  > ```java
  > DataSupport.daleteAll(Book.class,"price<?","15");
  > ```
  >
  > 意思是将数据库里的Book表中的数据price《15的删除

#### 使用LitePal查询数据

LitePal提供了很多非常有用的API。

1. DataSupport.findAll(Book.class);//查询Book表全部列，返回的是Book类型的List集合，即List<Book>

2. DataSupport.findFind(Book.class);//查询Book表中的第一行数据，返回的是Book 类型；

3. DataSupport.findLast(Book.class)l//查询Book表中的最后一行数据，返回的是Book类型；

4. DataSupport.select("name","author").find(Book.class);//在Book表中查询列为name和author的数据，返回的当然是List<Book>；

5. DataSupport.where("pages">?,"100").find("Book.class");//在Book表中查询pages大于100的数据；

6. DataSupport.order("price desc").find(Book.class);//将Book表中的Price按从大到小的顺序排序，desc表示

   ​                                                                                        // 降序,不写或asc表  示升序排列。

7. DataSupport.limit(3).find(Book.class);//这里指定了查询的数量为3，即只查询Book表中的前3条。

8. DataSupport.offset.limit(3).offset(1).find(Book.class);//这里表示查询Book表查询第2,3,4条数据；offset表示偏移量。

9. 当然，也可对上面的方法进行任意的连缀组合；

   > ```java
   >  List <Book> books = DataSupport().select("name","author")
   >                                   .where("pages>?","500")
   >                                   .oeder("pages").limit(10)
   >                                   .offset(10)
   >                                   .find(Book.class);
   > ```

