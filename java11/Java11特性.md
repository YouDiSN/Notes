# Java11

------

1. 更多的Unicode支持	

   ```java
   // java8
   public static void main(String[] args) {
       String str = "\uD83E\uDD2A";
       System.out.println(str);
   }
   
   // java11
   public static void main(String[] args) {
       String str = "🤪";
       System.out.println(str);
   }
   ```

   ![image-20181007150054138](/Users/xiwang/Documents/java学习笔记/笔记/Java/java11/Unicode字符支持.png)

   > **作用：**
   >
   > 可以在注释里面用Unicode表情吐槽产品经理的需求。。。
   >
   > [Unicode10链接](https://emojipedia.org)

2. 更好用的HttpClient

   如果用java自己原生的**URLConnection**样板代码比较多，而且性能也不是很好（每一次都要启动一个UrlConnection）。所以一般我们发http请求都是用的apache的HttpClient，因此需要一个依赖。

   但是java9的时候jdk源码新增了HttpClient。但是是处于实验状态。

   去年java9的演讲截图

   ![image-20181007201502898](/Users/xiwang/Documents/java学习笔记/笔记/Java/java11/java9HttpClient.png)

   java11正式发布了http包内的功能。

   java11发布的HttpClient除了以往我们一直用的阻塞型，还进一步支持响应式模型。

   就是JDK9加入的Flow。但是因为是java9才加入的，所以目前就是一套接口，以及一根独苗**SubmissionPublisher**。在java.util.concurrent包中暂时还没有提供其他更多成熟的响应式编程的组件。但是在http包中，已经有实现的仅供internal使用的（因为模块化，那些类虽然是public，但是并不能直接使用）响应式编程的相关组件。

   ```java
   package java.util.concurrent;
   
   public final class Flow {
   
       private Flow() {}
   
       @FunctionalInterface
       public static interface Publisher<T> {
           public void subscribe(Subscriber<? super T> subscriber);
       }
   
       public static interface Subscriber<T> {
           public void onSubscribe(Subscription subscription);
   
           public void onNext(T item);
   
           public void onError(Throwable throwable);
   
           public void onComplete();
       }
   
       public static interface Subscription {
           public void request(long n);
   
           public void cancel();
       }
   
       public static interface Processor<T,R> extends Subscriber<T>, Publisher<R> {
       }
   
       static final int DEFAULT_BUFFER_SIZE = 256;
   
       public static int defaultBufferSize() {
           return DEFAULT_BUFFER_SIZE;
       }
   
   }
   
   ```

3. 更安全的加密

   - 增强的KeyStore机制
   - JEP329 ChaCha20和Poly1305加密算法
   - 添加了Brainpool EC支持（RFC 5639）
   - Curve25519和Curve448的密钥协议
   - RSASSA-PSS签名支持已添加到SunMSCAPI
   - JEP332传输层安全性（TLS）1.3

4. lamda函数中可以使用var

   ```java
       public static void main(String[] args) {
           Map<String, String> stringMap = Map.of("k1", "v1", "k2", "v2", "k3", "v3");
   
           stringMap.forEach((k, v) -> {
               System.out.println(k);
               System.out.println(v);
           });
   
           // 这里可以用var，用var的目的是可以加注解
           stringMap.forEach((@NotEmpty var k, @Nullable var v) -> {
               System.out.println(k);
               System.out.println(v);
           });
       }
   ```

   lamda中使用var感觉只有在有一套完整的框架（这个框架会根据你的注解来做各种校验），这个框架的功能如果完善，那么这个功能就会比较强大，可以省去很多盘空等逻辑。如果没有这个框架，那么这个注解更多的只是标识的作用，那么感觉意义不是很大。

5. 基于嵌套的访问控制

   [JEP 181: Nest-Based Access Control](http://openjdk.java.net/jeps/181)

   引入一个机制，就是如果一些类逻辑上是属于同一个代码实体的，但是编译的时候被编译到了不同的文件中，那么这些类也应当共享私有变量等。

   动机主要是jvm上运行的语言（不仅仅是java），往往会把多个类放到同一个文件中（比如java 的嵌套类），这些类应当被认为是在“同一个类”中的。所以我们往往认为，access control（指的是private、java10开始的模块等控制）应当是共享的。之前为了实现这个功能，编译器往往需要将private字段的控制拓宽到package级别。然后在通过一个access bridges（控制桥接？）（这个东西我理解就是，通过生成一个packge可以访问级别的方法，来访问这些private变量）。

   这些桥接方法一方面增加了编译器的实现难度，一方面又增加了源码编译后的体积，同时也破坏了代码的封装性。

   新版本具体的实现就是新增一个nest级别的access control（public, protected, default（package）, private）。大家要注意的是这个是编译后的类文件中增加的，并不是我们代码中增加的级别。

   **需要注意的是，这个改动可能会影响到反射的API，以及MethodHandle句柄相关API，具体什么影响暂时还没研究出来。。。**

   这个改动还可以用来放置一些java jar包打包工具，在优化的时候因为内部类、嵌套类等逻辑，忽略了他们的顶部类。

6. 动态的类文件常量

   [JEP 309: Dynamic Class-File Constants](http://openjdk.java.net/jeps/309)

   拓展java类文件格式以支持一个新的常量池格式：**CONSTANT_Dynamic**。加载这个类文件变量也会在一个bootstrap方法中，就好比**invokedynamic**指令会在一个bootstrap方法中一样（lamda动态调用）。

   这个东西的主要目的是给lamda函数编译后的**invokedynamic**指令用的，为了实现这个指令，在常量池中添加了很多非常复杂的变量，也给相关的实现带了很高的复杂度，为了让这个功能更灵活强大，同时也提高性能和降低编译器的复杂性，就增加了这个机制。

7. GC：ZGC和Epsilon

   Epsilon就是一个什么都不敢的垃圾收集器。

   **ZGC垃圾收集器**，也称为**ZGC**，是一个可扩展的低延迟垃圾收集器，有如下特性：

   - 暂停时间**不**超过**10毫秒**
   - 暂停时间**不会**随堆或实时设置大小**而**增加
   - 处理堆范围从**几百M**到**几TB**。

8. shebang

   可以通过java文件头增加**#!/path/to/java --source version**，来直接这样调用java程序：**./test.java**。

   > 我自己试了。。好像不行— —#