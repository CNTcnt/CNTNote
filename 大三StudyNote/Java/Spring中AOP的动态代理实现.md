[TOC]

#  Spring 的 AOP 的实现原理

## 简介

* *AOP*为 Aspect Oriented Programming的缩写，意为：面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。
* 说起AOP就不得不说下OOP了，OOP中引入封装、继承和多态性等概念来建立一种对象层次结构，用以模拟公共行为的一个集合。但是，如果我们需要为部分对象引入公共部分的时候，OOP就会引入大量重复的代码。例如：日志功能。
* AOP技术利用一种称为“横切”的技术，解剖封装的对象内部，并将那些影响了多个类的公共行为封装到一个可重用模块，这样就能减少系统的重复代码，降低模块间的耦合度，并有利于未来的可操作性和可维护性。AOP把软件系统分为两个部分：核心关注点和横切关注点。业务处理的主要流程是核心关注点，与之关系不大的部分是横切关注点。横切关注点的一个特点是，他们经常发生在核心关注点的多处，而各处都基本相似。比如权限认证、日志、事务处理。

## 动态代理实现 AOP

* 在jdk1.3以后，jdk跟我们提供了一个API java.lang.reflect.InvocationHandler的类， 这个类可以让我们在JVM调用某个类的方法时动态的为些方法做些什么事。

* java 动态代理

  ~~~java
  //newProxyInstance，方法有三个参数：
  
  //loader: 用哪个类加载器去加载代理对象
  //interfaces:动态代理类需要实现的接口
  //h:动态代理对象方法在执行时，会调用h里面的invoke方法去执行
  
  public static Object newProxyInstance(ClassLoader loader,
                                            Class<?>[] interfaces,
                                            InvocationHandler h)
          throws IllegalArgumentException
  ~~~

* JDK的动态代理是靠多态和反射来实现的，它生成的代理类需要实现你传入的接口，并通过反射来得到接口的方法对象，并将此方法对象传参给增强类的invoke方法去执行（我们只需要实现增强类的 invoke 方法即可），从而实现了代理功能，故接口是jdk动态代理的核心实现方式，没有它就无法通过反射找到方法，所以这也是必须有接口的原因。

* ~~~java
  public class DynaProxyHello implements InvocationHandler{
        
       private Object target;//目标对象
        /**
         * 通过反射来实例化目标对象
         * @param object
         * @return
         */
        public Object bind(Object object){
           this.target = object;
           return Proxy.newProxyInstance(this.target.getClass().getClassLoader(), this.target.getClass().getInterfaces(), this);
       }
       
       @Override
       public Object invoke(Object proxy, Method method, Object[] args)
               throws Throwable {
           Object result = null;
           Logger.start();//添加额外的方法
           //通过反射机制来运行目标对象的方法
           result = method.invoke(this.target, args);
           Logger.end();
           return result;
       }
       
   }
  ~~~

  ~~~java
  
  //测试类代码
   public class DynaTest {
       public static void main(String[] args) {
           IHello hello = (IHello) new DynaProxyHello().bind(new Hello());//如果我们需要日志功能，则使用代理类
           //IHello hello = new Hello();//如果我们不需要日志功能则使用目标类
           hello.sayHello("明天");
       }
   }
  ~~~

* 但是有没有发现，我们上面的例子有问题，我们这个 DynaProxyHello 只有一个日志功能，如果我们需要权限验证功能呢？再创建一个 实现 InvocationHandler 的类？这样我们每个类的实现都依赖了底层实现,DynaPoxyHello对象和日志操作对象耦合太高，导致类的复用太差，违反了依赖倒置原则，所以我们可以把这些功能通过依赖注入进去，实现 依赖倒置

* 我们要在被代理对象的方法前面或者后面去加上日志操作代码(或者是其它操作的代码)，那么，我们可以抽象成一个接口，这个接口里就只有两个方法：一个是在被代理对象要执行方法之前执行的方法,我们取名为start，第二个方法就是在被代理对象执行方法之后执行的方法,我们取名为end。

  ~~~java
  //抽象成一个接口
  public interface ILogger {
       void start(Method method);
       void end(Method method);
  }
  //实现类
  public class DLogger implements ILogger{
        @Override
        public void start(Method method) {
            System.out.println(new Date()+ method.getName() + " say hello start...");
        }
        @Override
        public void end(Method method) {
            System.out.println(new Date()+ method.getName() + " say hello end");
        }
   }
  //现在可以把上面的动态代理类修改成
  public class DynaProxyHello1 implements InvocationHandler{
        //调用对象
        private Object proxy;
        //目标对象
        private Object target;
        
      //依赖注入进来
        public Object bind(Object target,Object proxy){
            this.target=target;
            this.proxy=proxy;
            return Proxy.newProxyInstance(this.target.getClass().getClassLoader(), this.target.getClass().getInterfaces(), this);
       }
       
       
       @Override
       public Object invoke(Object proxy, Method method, Object[] args)
               throws Throwable {
           Object result = null;
           //反射得到操作者的实例
           Class clazz = this.proxy.getClass();
           //反射得到操作者的Start方法
           Method start = clazz.getDeclaredMethod("start", new Class[]{Method.class});
           //反射执行start方法
           start.invoke(this.proxy, new Object[]{this.proxy.getClass()});
           //执行要处理对象的原本方法
           method.invoke(this.target, args);
           //反射得到操作者的end方法
           Method end = clazz.getDeclaredMethod("end", new Class[]{Method.class});
           //反射执行end方法
           end.invoke(this.proxy, new Object[]{method});
           return result;
       }
       
   }
  ~~~

  ~~~java
  //测试用例
  public class DynaTest1 {
       public static void main(String[] args) {
           IHello hello = (IHello) new DynaProxyHello1().bind(new Hello(),new DLogger());//如果我们需要日志功能，则使用代理类，把需要代理的对象和需要的功能对象注入进去
           //IHello hello = new Hello();//如果我们不需要日志功能则使用目标类
           hello.sayHello("明天");
       }
   }
  
  ~~~

* 通过上面例子，可以发现通过动态代理和发射技术，已经基本实现了AOP的功能，如果我们只需要在方法执行前打印日志，则可以不实现end()方法，这样就可以控制打印的时机了。如果我们想让指定的方法打印日志，我们只需要在invoke（）方法中加一个对method名字的判断，method的名字可以写在xml文件中，这样我们就可以实现以配置文件进行解耦了，这样我们就实现了一个简单的spring aop框架。