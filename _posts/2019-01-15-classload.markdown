---
layout: post
category: "Java"
title:  "JVM基础（二） 类加载过程"
tags: [Java, JVM, 源码]
---
* content
{:toc}

![](https://picsum.photos/800/300/?image=628)
Java中类的加载、连接都是在程序运行时完成的，虚拟机将class文件加载到内存，进行校验、解析和初始化最终形成可以被虚拟机使用的Java对象，即类加载过程。






## 类的生命周期
类在JVM的生命周期主要包括几个阶段：**加载**、**连接**、**初始化**、**使用**和**卸载**，其中连接部分又可细分为**验证**、**准备**和**解析**。

### 何时加载
加载时机并没有明确规定，由虚拟机自行掌握

### 何时初始化
JVM规定**有且只有**下面几种情况发生时会对类进行初始化，而加载、连接操作自然也在其之前完成。
- 读到下面四条指令：
    - `new` 创建一个对象实例
    - `getstatic` 读取一个类的静态变量（用final static修饰的静态变量除外，这部分会直接进入常量池）
    - `setstatic` 设置一个类的静态变量
    - `invokestatic` 调用一个类的静态方法
- 使用`java.lang.reflect`包里的方法进行反射调用
- 初始化一个类时，如果父类还没有初始化需要对父类进行初始化
- 用户指定的执行主类，虚拟机要对其进行初始化

## 加载
加载阶段完成下面几件事情：
- 通过类的全限定名获得类的二进制文件
- 将字节流所代表的静态存储结构转化为**方法区的运行时数据结构**
- 在**堆**中生成一个代表这个类的`java.lang.Class`对象，作为方法区数据的访问入口

### 类加载器
Java将类加载的工作交给JVM处理，既可以使用系统提供的类加载器进行加载，也可以用自定义的类加载器加载。
#### 分类
JVM提供了三个类加载器
- **启动类加载器 Bootstrap ClassLoader**：负责将存放在`<JAVA_HOME>\lib`目录下的或者被`-Xbootclasspath`指定的路径中的，可以被JVM识别的类库加载到虚拟机内存中。
> 启动类加载器不能被Java代码引用，会返回null。
- **扩展类加载器 Extension ClassLoader**：`<JAVA_HOME>\lib\ext`目录下，或者被`java.ext.dirs`系统变量指定的路径下的类。
- **系统类加载器 Application ClassLoader**：也叫应用程序类加载器，加载用户目录`classpath`下的类，是程序中默认的类加载器。

![](https://i.loli.net/2019/01/15/5c3d836b3d7df.png)

#### 双亲委派模型
对于任何一个类，需要**由类和它的加载器确定其在JVM的唯一性**，因此类加载需要依赖双亲委派模型。

除了最顶层的启动类加载器外，其他类加载器都应该有自己的父类加载器，通过组合实现。

双亲委派模型的具体工作模式是：**如果一个类加载器收到了类加载的请求，它不会先自己处理，而是委派给父类加载器进行处理，只有当福类加载器不能加载这个类（搜索范围内没有找到所需的类），子类才会尝试自己加载。**
![](https://i.loli.net/2019/01/15/5c3d836d111e9.png)

JVM中类都是通过`java.lang.ClassLoader`中的`loadClass()`方法进行加载的，源码通俗易懂，首先调用`findLoadedClass()`方法检查类是否已经被加载，之后调用父类的`loadClass()`方法，如果父类为null则调用Bootstrap类加载器进行加载，若还未加载成功则调用自己的`findClass`进行加载。
```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
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

## 连接
该阶段又可细分为验证、准备和解析。加载和连接阶段是交叉进行的，可能加载阶段还未结束，连接阶段就已经开始。

### 验证
保证class文件中包含的字节流符合JVM的格式要求，要进行下面几个check项：
- **文件格式验证**：是否符合class文件格式规范
- **元数据验证**：是否存在不符合java语义规范的元数据（对数据类型进行校验）
- **字节码验证**：对方法体进行校验，较为复杂这里不展开
- **符号引用验证**：验证符号引用是否可以转化为直接引用，为解析阶段做准备

### 准备
为**类变量**分配内存并设置默认值（零值，并非初始值），这些内存都是在**方法区**中分配的。（不包括用final static修饰的ConstantValue）

### 解析
将常量池中的**符号引用**翻译为**直接引用**的过程
- **符号引用**：以一组符号来描述所引用的目标，可以使任何字面值，引用的目标可能不在内存中
- **直接引用**：目标的指针、相对偏移量或能间接定位到目标的句柄，引用的目标必定存在于内存

解析主要包括**类或接口解析**、**字段解析**、**类方法解析**和**接口方法解析**，如果解析失败，即找不到对应的方法或字段则会抛出`java.lang.NoSuchMethodError`或`java.lang.NoSuchFieldError`。

## 初始化
初始化是执行**类构造器`<clinit>()`**方法的过程，这个方法有以下几个特点：
- 它是由编译器自动收集类中静态变量的赋值语句和静态语句块中的语句合并产生的，顺序取决于代码中的顺序。
- 不需要显式调用父类的该方法，JVM会保证在子类`<clinit>()`方法执行前，父类的该方法已经执行。
- 它不是必须的，如果没有静态变量和方法，就没有类构造器。
- JVM不会调用接口父类的该方法，接口类没有静态语句块。
- JVM会保证该方法在多线程环境下正确被加锁和同步。

(End)

>参考资料：
> - 《深入理解Java虚拟机》
