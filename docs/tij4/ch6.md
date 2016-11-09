## 第六章 访问权限控制
***
##### 1.访问权限控制的等级
* 从最大权限到最小权限依次为：public、protected、包访问权限（没有关键字）和private。

##### 2.包：库单元
Java用package关键字将构件捆绑到一个内聚的类库单元中。当编写一个Java源代码文件时，此文件通常被称为编译单元（有时也被称为转译单元）。每个编译单元必须有一个后缀名.java，而在编译单元内则可以有一个public类，该类的名称必须与文件的名称相同（包括大小写，但不包括文件的后缀名.java）。**每个编译单元只能有一个public类，否则编译器就不会接受。**如果在该编译单元之中还有额外的类的话，那么在包之外的世界是无法看到这些类的，这是因为它们不是public类，而且它们主要用来为主public类提供支持。
* package语句必须是文件中除注释以外的第一句程序代码。
* Java包的命名规则全部使用小写字母，包括中间的字也是如此。

>Java解释器的运行过程如下：  
>The Java interpreter proceeds as follows. First, it finds the environment variable CLASSPATH (set via the operating system, and sometimes by the installation program that installs Java or a Java-based tool on your machine). CLASSPATH contains one or more directories that are used as roots in a search for .class files. Starting at that root, the interpreter will take the package name and replace each dot with a slash to generate a path name off of the CLASSPATH root (so package foo.bar.baz becomes foo\bar\baz or foo/bar/baz or possibly something else, depending on your operating system). This is then concatenated to the various entries in the CLASSPATH. That’s where it looks for the .class file with the name corresponding to the class you’re trying to create. (It also searches some standard directories relative to where the Java interpreter resides.)

##### 3.Java访问权限修饰词
访问权限修饰词可以类中每个成员的定义之前（无论它是一个域还是一个方法）。
###### 包访问权限
* 默认访问权限没有任何关键字，但是通常是指包访问权限（有时也表示成为friendly）。其意味着当前的包中所有其他类对那个成员都有访问权限，但是对于这个包之外的所有类，这个成员却是**private**。由于一个编译单元（即一个文件），只能隶属于一个包，所有经由包访问权限，处于同一个编译单元中的所有类彼此之间都是自动可访问的。

取得对某个成员的访问权的唯一途径是：  
![取得访问权限的方法](/images/posts/tij_access00.png)  
**注意：**同处于相同的目录但是没有给自己设定任何包名称的Java文件将被看做是隶属于该目录的默认包之中。
>All the files within the same directory that don’t have explicit package declarations are implicitly part of the default package for
that directory.

###### public:接口访问权限
使用关键字public，就意味着public之后紧跟着的成员声明自己对每个人（任何类）都是可用的（可访问的），尤其是使用类库的客户程序员更是如此。

###### private：你无法访问
关键字private的意思是，除了包含该成员的类之外，其他任何类都无法访问这个成员。  
**注意：**使用类的客户端程序员是无法访问包访问权限成员的。
private的运用示例：可能想控制如何创建对象，并阻止别人直接访问某个特定的构造器（或全部构造器）。示例如下：  
```Java
class Sundae {
  private Sundae() {}
  static Sundae makeASundae() {
    return new Sundae();
  }
}

public class IceCream {
  public static void main(String[] args) {
    //! Sundae x = new Sundae();
    Sundae x = Sundae.makeASundae();
  }
}
```
上述示例中，不能通过构造器来创建Sundae对象，而必须调用makeASundae()方法来达到此目的。  
* **说明：**除非必须公开底层实现细目（此种境况很少），否则就应该将所有的域指定为private。
* However, just because a reference to an object is private inside a class doesn’t mean that some other object can’t have a public reference to the same object.（如何理解此句话？）

###### protected：继承访问权限
* 如果创建了一个新包，并在该新包中创建一个类（子类）继承另一个包中的某个类（基类），那么唯一可以访问基类的成员就是其public成员和protected成员（当然，如果在同一个包内执行继承工作，就可以操纵所有的拥有包访问权限的成员）。
* protected也提供包访问权限，即相同包内的其他所有类可以访问protected元素。

###### 小结：访问权限示意图

![访问权限示意图](/images/posts/tij_access01.png)  

##### 4.接口和实现
* 访问权限的控制常被称为是具体实现的隐藏。把数据和方法包装进类中，以及具体实现的隐藏，行共同被称作是封装。其结果是一个同时带有特征和行为的数据类型。  
* 为了清楚起见，可能会采用一种将public成员置于开头，后面紧跟protected、包访问权限和private成员的创建类形式。示例如下：  
```Java
public class OrganizedByAccess {
  public void pub1() { /* ... */ }
  public void pub2() { /* ... */ }
  public void pub3() { /* ... */ }
  private void priv1() { /* ... */ }
  private void priv2() { /* ... */ }
  private void priv3() { /* ... */ }
  private int i;
  // ...
}
```

##### 5.类的访问权限
* 为了控制某个类的访问权限，修饰词必须出现于关键字class之前，如，`public class Widget{...}`。  
类的一些限制如下：
![类的限制](/images/posts/tij_access02.png)  
* 类既不可以是private(否则，除该类外，其他任何类都不可以访问它)，也不可以是protected的。所以对类的访问权限，仅有两个选择：包访问权限或public。如果不希望其他人对该类用于访问权限，可以把所以的构造器都指定为private，从而阻止任何人创建该类的对象，但是有一个例外，就是你在该类的static成员内部可以创建该类的对象。示例如下：

```Java
class Soup1 {
  private Soup1() {}
  // (1) Allow creation via static method:
  public static Soup1 makeSoup() {
    return new Soup1();
  }
}

class Soup2 {
  private Soup2() {}
  // (2) Create a static object and return a reference
  // upon request.(The "Singleton" pattern):
  private static Soup2 ps1 = new Soup2();
  public static Soup2 access() {
    return ps1;
  }
  public void f() {}
}

// Only one public class allowed per file:
public class Lunch {
  void testPrivate() {
    // Can't do this! Private constructor:
    //! Soup1 soup = new Soup1();
  }
  void testStatic() {
    Soup1 soup = Soup1.makeSoup();
  }
  void testSingleton() {
    Soup2.access().f();
  }
}
```
**说明**：  
1）如果想要在返回引用前在Soup1上做一些额外的工作，或是如果想要记录到底创建了多少个Soup1对象（可能与限制其数量），这种做法将会大有裨益的。  
2）Soup2用到了单例模式（singleton），这是因为我们始终只能创建它的一个对象。  
* **注意：**  
1）如果没有为类访问权限指定一个访问修饰符，它就会默认得到包访问权限。此时，即使将该类的构造器声明为public的，也不能在包外实例化该类。并且，即使该类中拥有static public字段或static public方法，也不能在包外直接通过该类访问。总之，在具有包访问权限的类中，使用public并不能将其修饰的成员或构造器提升到public权限，即使用public无效。  

	2）一个类的默认构造器的访问权限和该类的访问权限相同。  

	3）==内部类可以是private或protected的。==






















































