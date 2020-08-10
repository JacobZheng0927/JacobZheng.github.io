---
layout: post
title: '深入学习Java虚拟机：第3讲'
subtitle: 'Spark基础概念介绍'
date: 2020-08-10
categories: 技术
cover: ''
tags: java jvm classLoader
---

# ClassLoader与类加载

## 一、类基础

> java特性：跨平台，一次编译，到处运行
>

一个JAVA类从编写到使用，会经过以下流程

```mermaid
graph LR
file[.java文件]--编译-->cla[.class文件]
cla--不同平台JVM解析-->command[机器指令]
```

先编译成字节码，再由不同平台JVM解析，运行时不需要重编译。java虚拟机在执行字节码时，转换成机器指令。 

> 为什么不解析成机器码？
>
> 1. 不用每次执行需要检查 
> 2. 保持兼容性 例如scala



## 二、类加载

由上一节可知，JVM需要使用一个类，需要现有一个将.class文件加载到内存中的过程



### 1. 类加载的时机

类从被加载到JVM中到卸载为止，生命周期分为以下7个阶段。

![1595409-20190521154930506-891623513](https://markdown-image-upload.oss-cn-beijing.aliyuncs.com/img/1595409-20190521154930506-891623513.png)



图中 加载、验证、准备、初始化、卸载这5个阶段的顺序是确定的，解析和使用不一定。

虚拟机规范规定，只有5种情况必须立即对类进行初始化（即加载、验证、准备均已完成）。

> - 遇到`new`、`getstatic`、`putstatic`、`invokestatic`这4条指令时，如果类没有初始化需要先触发初始化。对应Java代码场景，则是**使用new关键字实例化对象**时，**读取或设置一个类的静态字段时**（被final修饰、已在编译器把结果放入常量池的静态字段除外）以及**调用一个类的静态方法的时候**。
> - 使用java.lang.reflect包的方法对类进行反射调用的时候，如果类没有初始化需要先触发其初始化。
> - 当初始化一个类时，其父类没有初始化，先初始化其父类
> - 虚拟机启动时，指定的执行的主类先初始化
> - java.lang.invoke.MethodHandle实例最后解析结果为`REF_getstatic`、`REF_putstatic`、`REF_invokestatic`的方法句柄，且这个方法句柄对应的类没有初始化过，需要先触发初始化。

以上5种情况称为主动引用，其他的引用类的方式都是被动引用，不会触发初始化。

```java
/**
 * 被动使用类字段演示一：
* 通过子类引用父类的静态字段，不会导致子类初始化
 **/
public class SuperClass {
	static {
		System.out.println("SuperClass init!");
	}
	public static int value = 123;
}

public class SubClass extends SuperClass {
	static {
		System.out.println("SubClass init!");
	}
}

/**
 * 非主动使用类字段演示
 * 输出SuperClass init! 子类没有初始化
 **/
public class NotInitialization {
	public static void main(String[] args) {
		System.out.println(SubClass.value);
	}
}

/**
 * 被动使用类字段演示二：
* 通过数组定义来引用类，不会触发此类的初始化
 **/
public class NotInitialization {
	public static void main(String[] args) {
		SuperClass[] sca = new SuperClass[10];
	}
}
```



```java
/**
 * 被动使用类字段演示三：
 * 被final修饰
 * 常量在编译阶段会存入调用类的常量池中，本质上没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。
 **/
public class ConstClass {
	static {
		System.out.println("ConstClass init!");
	}
	public static final String HELLOWORLD = "hello world";
}

/**
 * 非主动使用类字段演示
 **/
public class NotInitialization {
	public static void main(String[] args) {
		System.out.println(ConstClass.HELLOWORLD);
	}
}
```



### 2. 类加载的过程

> 即加载、验证、准备、解析、初始化这5个阶段

- 加载

  加载需要完成三件事：

  1. 通过类的全限定名获取定义类的二进制字节流

     可以从各种来源读取，例如zip包，网络，运行时计算生成（动态代理）...

  2. 将字节流所代表的的静态存储结构转化为方法区运行时数据结构

  3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区该类的各种数据访问入口



- 验证

  1. 文件格式验证（字节流是否符合Class文件格式规范）
  2. 元数据验证（元数据语义验证）
  3. 字节码验证（校验类在运行时不会危害虚拟机）
  4. 符号引用验证（对类以外的信息进行匹配性校验）

  

- 准备

  为类分配内存并设置变量初始值

  

- 解析

  将符号引用替换为直接引用，确定引用目标

  

- 初始化

  开始执行类中定义的Java代码程序



### 3. 类加载器

#### 3.1 类与类加载器

> 对于任意一个类，都需要由他的类加载器和这个类本身一通确立其在Java虚拟机的唯一性，每一个类加载器，都拥有一个独立的类名称空间。

#### 3.2 ClassLoader结构

![java类加载机制](https://markdown-image-upload.oss-cn-beijing.aliyuncs.com/img/java%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.png)

所有类加载器的基类，它是抽象的，定义了类加载最核心的操作。所有继承与classloader的加载器，都会优先判断是否被父类加载器加载过，防止多次加载，防止加载冲突

> 二进制名称形如
>
> `java.lang.String`
>
> `javax.swing.JSpinner$DefaultEditor`
>
> `java.security.KeyStore$Builder$FileBuilder$1`
>
> `java.net.URLClassLoader$3$1`

```java
    /**
		 * 用指定的二进制名称加载类。
     *
     * @param  name
     *         The <a href="#name">binary name</a> of the class
     *
     * @return  The resulting <tt>Class</tt> object
     *
     * @throws  ClassNotFoundException
     *          If the class was not found
     */
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }

		protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
    		//锁，防止多次加载，所以jvm启动巨慢
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                		//准备委派给父类加载
                    if (parent != null) {
                    		//父类存在，委派给父类
                        c = parent.loadClass(name, false);
                    } else {
                    		//父类不存在，委派给启动类加载器
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

	            if (c == null) {
	               // If still not found, then invoke findClass in order
	               // to find the class.父类加载不到，自身再加载
	               long t1 = System.nanoTime();
	               c = findClass(name);

	               // this is the defining class loader; record the stats
	               sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
	               sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
	               sun.misc.PerfCounter.getFindClasses().increment();
	           }
	       }
	       if (resolve) {
         		resolveClass(c);
         }
	        return c;
	   }
}
```

jdk 1.7为了提供并行加载class，提供ClassLoader.ParallelLoaders内部类，用来封装一组并行能力的加载器类型。这个一般是用不到的，有兴趣可以先看一下。但是需要知道ClassLoader是支持并行加载的。



---

- Bootstrap classLoader

位于java.lang.classload，所有的classload都要经过这个classload判断是否已经被加载过，采用native code实现，是JVM的一部分，主要加载JVM自身工作需要的类，如java.lang.、java.uti.等； 这些类位于$JAVA_HOME/jre/lib/rt.jar。Bootstrap ClassLoader不继承自ClassLoader，因为它不是一个普通的Java类，底层由C++编写，已嵌入到了JVM内核当中，当JVM启动后，Bootstrap ClassLoader也随着启动，负责加载完核心类库后，并构造Extension ClassLoader和App ClassLoader类加载器。

```java
/**
 * Returns a class loaded by the bootstrap class loader;
 * or return null if not found.
 */
private Class<?> findBootstrapClassOrNull(String name)
{
    if (!checkName(name)) return null;

    return findBootstrapClass(name);
}

// return null if not found
private native Class<?> findBootstrapClass(String name);
```



---

- SecureClassLoader

继承自ClassLoader，添加了关联类源码、关联系统policy权限等支持。



---

- URLClassLoader

继承自SecureClassLoader，支持从jar文件和文件夹中获取class，继承于classload，加载时首先去classload里判断是否由bootstrap classload加载过，1.7 新增实现closeable接口，实现在try 中自动释放资源，但扑捉不了.close()异常

```java
/**
 * This class loader is used to load classes and resources from a search
 * path of URLs referring to both JAR files and directories. Any URL that
 * ends with a '/' is assumed to refer to a directory. Otherwise, the URL
 * is assumed to refer to a JAR file which will be opened as needed.
 * <p>
 * The AccessControlContext of the thread that created the instance of
 * URLClassLoader will be used when subsequently loading classes and
 * resources.
 * <p>
 * The classes that are loaded are by default granted permission only to
 * access the URLs specified when the URLClassLoader was created.
 *
 * @author  David Connelly
 * @since   1.2
 */
public class URLClassLoader extends SecureClassLoader implements Closeable
```



---

- ExtClassLoader

扩展类加载器，继承自URLClassLoader继承于urlclassload，扩展的class loader，加载位于$JAVA_HOME/jre/lib/ext目录下的扩展jar。查看源码可知其查找范围为System.getProperty(“java.ext.dirs”)。

```java
public static Launcher.ExtClassLoader getExtClassLoader() throws IOException {
            //System.getProperty("java.ext.dirs");
            //在项目启动时就加载所有的ext.dirs目录下的文件，并将其初始化
            final File[] var0 = getExtDirs();

	        try {
	        //AccessController.doPrivileged特权，让程序突破当前域权限限制，临时扩大访问权限
	           return (Launcher.ExtClassLoader)AccessController.doPrivileged(new PrivilegedExceptionAction<Launcher.ExtClassLoader>() {
	               public Launcher.ExtClassLoader run() throws IOException {
	                   int var1 = var0.length;

	                   for(int var2 = 0; var2 < var1; ++var2) {
	                       MetaIndex.registerDirectory(var0[var2]);
	                   }

	                   return new Launcher.ExtClassLoader(var0);
	               }
	           });
	       } catch (PrivilegedActionException var2) {
	           throw (IOException)var2.getException();
	       }
	   }
```



---

- AppClassLoader

应用类加载器，继承自URLClassLoader，也叫系统类加载器（ClassLoader.getSystemClassLoader()可得到它），它负载加载应用的classpath下的类，查找范围System.getProperty(“java.class.path”)，通过-cp或-classpath指定的类都会被其加载，没有完全遵循双亲委派模型的，它重写的是loadClass方法

```java
public Class<?> loadClass(String var1, boolean var2) throws ClassNotFoundException {
    int var3 = var1.lastIndexOf(46);
    if (var3 != -1) {
        SecurityManager var4 = System.getSecurityManager();
        if (var4 != null) {
            var4.checkPackageAccess(var1.substring(0, var3));
        }
    }

  	//ucp是SharedSecrets获取的Java栈帧中存储的类信息
    if (this.ucp.knownToNotExist(var1)) {
      	//顶级类classloader加载的信息
        Class var5 = this.findLoadedClass(var1);
        if (var5 != null) {
            if (var2) {
							//link过程，Class载入必须link，link指的是把单一的Class加入到有继承关系的类树中
                this.resolveClass(var5);
            }

            return var5;
        } else {
            throw new ClassNotFoundException(var1);
        }
    } else {
        return super.loadClass(var1, var2);
    }
}
```



---

- Launcher

java程序入口，负责实例化相关class，ExtClassLoader和AppClassLoader都是其内部实现类



#### 3.3 双亲委派模型

ClassLoader使用的是双亲委派机制来搜索加载类的，每个ClassLoader实例都有一个父类加载器的引用（不是继承的关系，是一个组合的关系），虚拟机内置的类加载器（Bootstrap ClassLoader）本身没有父类加载器，但可以用作其它ClassLoader实例的的父类加载器。当一个ClassLoader实例需要加载某个类时，它会试图亲自搜索某个类之前，先把这个任务委托给它的父类加载器，这个过程是由上至下依次检查的，首先由最顶层的类加载器Bootstrap ClassLoader试图加载，如果没加载到，则把任务转交给Extension ClassLoader试图加载，如果也没加载到，则转交给App ClassLoader 进行加载，如果它也没有加载得到的话，则返回给委托的发起者，由它到指定的文件系统或网络等URL中加载该类。如果它们都没有加载到这个类时，则抛出ClassNotFoundException异常。否则将这个找到的类生成一个类的定义，并将它加载到内存当中，最后返回这个类在内存中的Class实例对象。

![类加载过程](https://markdown-image-upload.oss-cn-beijing.aliyuncs.com/img/%E7%B1%BB%E5%8A%A0%E8%BD%BD%E8%BF%87%E7%A8%8B.png)



- 如何破坏双亲委派模型

  1. 重写loadClass()

  2. 线程上下文类加载器(Thread Context ClassLoader)。这个类加载器可以通过java.lang.Thread类的setContextClassLoader()方法进行设置，如果创建线程时还未设置，他将会从父线程中继承一个，如果在应用程序的全局范围内都没有设置过的话，那这个类加载器默认就是应用程序类加载器。

     有了线程上下文加载器，JNDI服务就可以使用它去加载所需要的SPI代码，也就是父类加载器请求子类加载器去完成类加载的动作，这种行为实际上就是打通了双亲委派模型层次结构来逆向使用类加载器，实际上已经违背了双亲委派模型的一般性原则，但这也是无可奈何的事情。Java中所有涉及SPI的加载动作基本上都采用这种方式，例如JNDI、JDBC、JCE、JAXB和JBI等。

     eg：JDBC驱动加载

     - 不破坏双亲委派的情况

       ```java
       // 1.加载数据访问驱动
       Class.forName("com.mysql.jdbc.Driver");
       //2.连接到数据"库"上去
       Connection conn= DriverManager.getConnection("jdbc:mysql://localhost:3306/mydb？characterEncoding=GBK", "root", "");
       ```

       核心就是这句Class.forName()触发了mysql驱动的加载，我们看下mysql对Driver接口的实现：

       ```java
       public class Driver extends NonRegisteringDriver implements java.sql.Driver {
           public Driver() throws SQLException {
           }
       
           static {
               try {
                   DriverManager.registerDriver(new Driver());
               } catch (SQLException var1) {
                   throw new RuntimeException("Can't register driver!");
               }
           }
       }
       ```

       可以看到，Class.forName()其实触发了静态代码块，然后向DriverManager中注册了一个mysql的Driver实现。

     - 破坏双亲委派的情况

       在JDBC4.0以后，开始支持使用spi的方式来注册这个Driver，具体做法就是在mysql的jar包中的`META-INF/services/java.sql.Driver `文件中指明当前使用的Driver是哪个，然后使用的时候就直接这样就可以了：

       ```bash
        Connection conn= DriverManager.getConnection("jdbc:mysql://localhost:3306/mydb?characterEncoding=GBK", "root", "");
       ```

       可以看到这里直接获取连接，省去了上面的Class.forName()注册过程。
        现在，我们分析下看使用了这种spi服务的模式原本的过程是怎样的:

       - 第一，从META-INF/services/java.sql.Driver文件中获取具体的实现类名“com.mysql.jdbc.Driver”
       - 第二，加载这个类，这里肯定只能用class.forName("com.mysql.jdbc.Driver")来加载

       好了，问题来了，Class.forName()加载用的是调用者的Classloader，这个调用者DriverManager是在rt.jar中的，ClassLoader是启动类加载器，而com.mysql.jdbc.Driver肯定不在<JAVA_HOME>/lib下，所以肯定是无法加载mysql中的这个类的。这就是双亲委派模型的局限性了，父级加载器无法加载子级类加载器路径中的类。

       那么，这个问题如何解决呢？按照目前情况来分析，这个mysql的drvier只有应用类加载器能加载，那么我们只要在启动类加载器中有方法获取应用程序类加载器，然后通过它去加载就可以了。这就是所谓的线程上下文加载器。
        **线程上下文类加载器可以通过Thread.setContextClassLoaser()方法设置，如果不特殊设置会从父类继承，一般默认使用的是应用程序类加载器**

       **很明显，线程上下文类加载器让父级类加载器能通过调用子级类加载器来加载类，这打破了双亲委派模型的原则**

       现在我们看下DriverManager是如何使用线程上下文类加载器去加载第三方jar包中的Driver类的。

       ```java
       public class DriverManager {
           static {
               loadInitialDrivers();
               println("JDBC DriverManager initialized");
           }
           private static void loadInitialDrivers() {
               //省略代码
               //这里就是查找各个sql厂商在自己的jar包中通过spi注册的驱动
               ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
               Iterator<Driver> driversIterator = loadedDrivers.iterator();
               try{
                    while(driversIterator.hasNext()) {
                       driversIterator.next();
                    }
               } catch(Throwable t) {
                       // Do nothing
               }
       
               //省略代码
           }
       }
       ```

       使用时，我们直接调用DriverManager.getConn()方法自然会触发静态代码块的执行，开始加载驱动
        然后我们看下ServiceLoader.load()的具体实现：

       ```php
           public static <S> ServiceLoader<S> load(Class<S> service) {
               ClassLoader cl = Thread.currentThread().getContextClassLoader();
               return ServiceLoader.load(service, cl);
           }
           public static <S> ServiceLoader<S> load(Class<S> service,
                                                   ClassLoader loader){
               return new ServiceLoader<>(service, loader);
           }
       ```

       可以看到核心就是拿到线程上下文类加载器，然后构造了一个ServiceLoader,后续的具体查找过程，我们不再深入分析，这里只要知道这个ServiceLoader已经拿到了线程上下文类加载器即可。
        接下来，DriverManager的loadInitialDrivers()方法中有一句**driversIterator.next();**,它的具体实现如下：

       ```csharp
       private S nextService() {
                   if (!hasNextService())
                       throw new NoSuchElementException();
                   String cn = nextName;
                   nextName = null;
                   Class<?> c = null;
                   try {
                       //此处的cn就是产商在META-INF/services/java.sql.Driver文件中注册的Driver具体实现类的名称
                      //此处的loader就是之前构造ServiceLoader时传进去的线程上下文类加载器
                       c = Class.forName(cn, false, loader);
                   } catch (ClassNotFoundException x) {
                       fail(service,
                            "Provider " + cn + " not found");
                   }
                //省略部分代码
               }
       ```

       现在，我们成功的做到了通过线程上下文类加载器拿到了应用程序类加载器（或者自定义的然后塞到线程上下文中的），同时我们也查找到了厂商在子级的jar包中注册的驱动具体实现类名，这样我们就可以成功的在rt.jar包中的DriverManager中成功的加载了放在第三方应用程序包中的类了。

       这个时候我们再看下整个mysql的驱动加载过程:

       - 第一，获取线程上下文类加载器，从而也就获得了应用程序类加载器（也可能是自定义的类加载器）
       - 第二，从META-INF/services/java.sql.Driver文件中获取具体的实现类名“com.mysql.jdbc.Driver”
       - 第三，通过线程上下文类加载器去加载这个Driver类，从而避开了双亲委派模型的弊端

       很明显，mysql驱动采用的这种spi服务确确实实是破坏了双亲委派模型的，毕竟做到了父级类加载器加载了子级路径中的类。

  3. OSGi实现模块化热部署