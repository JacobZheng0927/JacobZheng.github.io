# ClassLoader与类加载

## 1.类基础

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



## 2.类加载

由上一节可知，JVM需要使用一个类，需要现有一个将.class文件加载到内存中的过程



### 2.1 类加载的时机

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



### 2.2 类加载的过程

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



### 2.3 类加载器

#### 类与类加载器

> 对于任意一个类，都需要

## ClassLoader结构

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

### Bootstrap classLoader

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

### SecureClassLoader

继承自ClassLoader，添加了关联类源码、关联系统policy权限等支持。

### URLClassLoader

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

### ExtClassLoader

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

### AppClassLoader

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

### Launcher

java程序入口，负责实例化相关class，ExtClassLoader和AppClassLoader都是其内部实现类

## 双亲加载机制

ClassLoader使用的是双亲委托机制来搜索加载类的，每个ClassLoader实例都有一个父类加载器的引用（不是继承的关系，是一个组合的关系），虚拟机内置的类加载器（Bootstrap ClassLoader）本身没有父类加载器，但可以用作其它ClassLoader实例的的父类加载器。当一个ClassLoader实例需要加载某个类时，它会试图亲自搜索某个类之前，先把这个任务委托给它的父类加载器，这个过程是由上至下依次检查的，首先由最顶层的类加载器Bootstrap ClassLoader试图加载，如果没加载到，则把任务转交给Extension ClassLoader试图加载，如果也没加载到，则转交给App ClassLoader 进行加载，如果它也没有加载得到的话，则返回给委托的发起者，由它到指定的文件系统或网络等URL中加载该类。如果它们都没有加载到这个类时，则抛出ClassNotFoundException异常。否则将这个找到的类生成一个类的定义，并将它加载到内存当中，最后返回这个类在内存中的Class实例对象。

![类加载过程](https://markdown-image-upload.oss-cn-beijing.aliyuncs.com/img/%E7%B1%BB%E5%8A%A0%E8%BD%BD%E8%BF%87%E7%A8%8B.png)