[TOC]

***

#### 使用Intent传递对象

- Intent的用法有很多，我们可以借助它来启动活动，发送广播，启动服务等；在上述操作中，我们还可以添加一些附加数据，以达到传值的效果；

- 当我们要传普通数据类型的时候，我们是使用

  ~~~java
  Intent intent = new Intent(FirstActivity.this,SecondActivity.this);
  intent.putExtra("string_data","hello");
  intent.putExtra("int_data",100);
  startActivity(intent);
  ~~~

  在使用时

  ~~~java
  getIntent().getStringExtra("string_data");
  getIntent().getIntExtra("int_data"，0);
  ~~~

- 但当我们需要传递自定义对象的时候，上面这种方式就力不从心，那么有2种方式来实现我们需要的传递自定义对象,一种是Serializable(序列化)方式，一种是Parcelable(打包)方式；

##### Serializable方式（序列化）

- Serializable会将整个对象序列化进行传输，序列化的对象可以在网络上传输，也可以在储存在本地；但是效率会比Parcelable低一些；

- 实现Serializable方式也很简单，只需要实现Serializable接口即可；

  ~~~java
  public class People implements Serializable {//最重要的是实现这个接口的类的对象都是可序列化的
      private String name;
      private int age;
      public String getName() {
          return name;}
      public void setName(String name) {
          this.name = name;}
      public int getAge() {
          return age;}
      public void setAge(int age) {
          this.age = age; }
  ~~~

  利用Intent传输时，也十分简单

  ~~~java
  People people = new People();
  people.setName("Tom");
  people.setAge(20);
  Intent intent = new Intent(FirstActivity.this,SecondActivity.this);
  intent.putExtra("people_data",people);
  startActivity(intent);
  ~~~

  利用Intent获取信息

  ~~~java
  People people = (People)getIntent().getSerializableExtra("people_data");
  ~~~

##### Parcelable方式（打包）

- 通常情况下推荐使用这种方式；

~~~java

public class Person implements Parcelable {//首先创建一个类实现Parcelable接口；
    private String name;//把要传输的数据列出
    private int age;
    @Override//必须重写2个方法
    public int describeContents() {//这个方法返回0即可；
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {//这个方法要将数据属性一一写入
        dest.writeString(name);
        dest.writeInt(age);
    }
  //这个类中必须要创建一个CREATOR对象，这个对象是一个Parcelable.Creator的实现，并将泛型指定为Person型
    public static final Parcelable.Creator<Person> CREATOR = new Parcelable.Creator<Person>(){
        @Override//创建这个匿名类对象需要重写2个方法
        public Person createFromParcel(Parcel source) {
            Person person = new Person();//new一个传输对象
            person.name = source.readString();//读取顺序要和写入顺序一致
            person.age = source.readInt();
            return person;
        }
        @Override
        public Person[] newArray(int size) {
            return new Person[size];//new出一个传输对象size大小的数组，
        }
    };
~~~

~~~java
传输对象时和以前同样使用，只是在获取对象时有点不同
Person person = (Person)getIntent().getParcelableEtra("person_data");
~~~

