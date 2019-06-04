[TOC]



# Builder模式

## 介绍

* Builder模式时一步一步创建一个复杂对象的模式，允许用户在不知道内部构建细节的情况下，可以更精细地控制对象的构造流程；

* 该模式为了将构建复杂对象的过程和它的部件解耦，使得构建过程和部件的表示隔离开发；

* 比如一个对象有大量的组成部分，这些部分是在创建时动态添加的，如创建一辆汽车，要有方向盘，发动机，还有各种小零件，如何去把这些零件组装成一辆车，这个装配过程很复杂，但是！！**我并不想关心创建过程，我只想给你几个零件，然后，你给我一辆车**，此时，对于这种情况，为了在构建过程中对外部隐藏实现细节，就可以使用Builder模式将部件和组装过程分离，用户只需关心部件而不需要去关心这些部件怎么组装，使得构建过程和部件都可以自由扩展，两者之间的耦合降到最低；

* 目的对象的创建依靠其内部类 目的类.Builder，使用一种方法链的方式来创建；如下面的例子Student对象依靠其内部类Student.Builder构建；

* 以下一个例子复制李嘉豪的（比较容易理解）

* > 你有一个全世界人都羡慕的工作，你要记录大熊猫的身体健康状态，身高，这顿吃了没有，有没有男朋友女朋友，然后将这些数据给需要的工作人员。其实你的工作就是记录数据到对应的表中对吧，然后把这份数据表给被人吧
  >
  > 抽象点来看，你就是那个Builder，别人想要熊猫的数据，找你要就好了，具体怎么记录的数据，是你这个builder的事，和别人无关

  ## 

## 使用场景

1. 当初始化一个对象特别复杂，如参数多，且很多参数具有默认值时；
2. 当一个对象的创建可以由若干个部件装配时，（即你给1个部件也可以创建对象，2个部件也可以，但是，这2个对象内部是不同的）
3. 当一个对象非常复杂，或者对象的调用顺序不同产生了不同的作用；
4. 比如你写一个类，类似一个JavaBean，里面用来存数据，在设置这些数据就可以用到Builder

## 优缺点

* 优点：构建过程和构建的部件的耦合低；代码可读性较高

  ​	   避免了众多的setting()方法；

  ​	   建造者独立，易扩展；

  ​	   良好的封装性，可以使得客户端不必知道产品内部如何构建；

  ​	   将一个复杂对象的构建与它的表示分离，即用Builder模式来构建一个不可变的配置对象，并且将这个配置对象		     注入到目的对象中，也就是说，它只能在构建时设置各个参数，一旦调用类build()方法，构建对象之后，它的属性就不可再修改，因为目的对象它没有了setting()方法，字段也都隐藏，此时用户可见的函数减少，代码的使用成本降低

* 缺点：会产生多余的Builder对象以及Director对象，消耗内存，代码比较冗长

## 举例

~~~java
public class Student {

    private int id;
    private String name;
    private String passwd;
    private String sex;
    private String address;
    // 构造器尽量缩小范围
    private Student() {
    }
    // 构造器尽量缩小范围
    private Student(Student origin) {
        // 拷贝一份
         this.id = origin.id;
        this.name = origin.name;
        this.passwd = origin.passwd;
        this.sex = origin.sex;
        this.address = origin.address;
    }
    public int getId() {
        return id;
    }
    public String getName() {
        return name;
    }
    public String getPasswd() {
        return passwd;
    }
    public String getSex() {
        return sex;
    }
    public String getAddress() {
        return address;
    }

    /**
     * Student的创建完全依靠内部类Student.Builder，使用一种方法链的方式来创建
     *
     */
    public static class Builder {
        private Student target;
        public Builder() {
            target = new Student();
        }
        public Builder setId(int id) {
            target.id = id;
            return this;
        }
        public Builder setName(String name) {
            target.name = name;
            return this;
        }
        public Builder setPassword(String passwd) {
            target.passwd = passwd;
            return this;
        }
        public Builder setSex(String sex) {
            target.sex = sex;
            return this;
        }
        public Builder setAddress(String address) {
            target.address = address;
            return this;
        }
        public Student build() {
            return new Student(target);
        }       
    }
}
//客户端实现
Student s = new Student.Builder()
  					 .setName("CC")
  					 .setPassword("qwerty")
  					 .setSex("男")
  					 .setAddress("银河系第二旋臂")
  					 .build();
~~~

