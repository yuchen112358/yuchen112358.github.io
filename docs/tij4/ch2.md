## Java编程思想（第四版）学习笔记
#### By 王震 <wzzju@mail.ustc.edu.cn>
***
### 第二章 一切都是对象
1\.遥控器（引用）和电视机（对象）

2\.安全做法：创建一个引用的同时便进行初始化。如：`String str = "asdf"`
或`String str=new String("asdf");`。

3\.Java中，对象引用存储在堆栈（stack）中，但是Java对象并不存储于其中。

4\.堆（heap）用于存放所有的Java对象。

5\.stack和heap都位于RAM区。

6\.常量存储在代码内部。

7\.特例：基本类型不用new来创建变量，而是创建一个并非是引用的“自动”变量，这个变量直接存储“值”，并置于堆栈中（这样会更加高效）。

8\.Java的基本类型：

|基本类型|大小|包装器类型|
|:---|:---:|---:|
|boolean|-|Boolean|
|char|16-bit|Character|
|byte|8 bits|Byte|
|short|16 bits|Short|
|int|32 bits|Integer|
|long|64 bits|Long|
|float|32 bits|Float|
|double|64 bits|Double|
|void|-|Void|

9\.所有数值类型都有正负号，故不要去寻找无符号的数值类型。
注意：byte的取值为`Unicode 0`到`Unicode 2^16^-1`。

10\.boolean类型所占存储空间的大小没有明确指定，仅定义为能够取字面值true或false。

11\.基本类型具有的包装器类，使得可以在堆中创建一个非基本对象，用来表示对应的基本类型。

12\.Java提供了两个用于高精度计算的类：BigInteger和BigDecimal。虽然它们大体上属于“包装器类”的范畴，但是二者都没有对应的基本类型。必须以方法调用方式取代运算符方式来实现运算操作==（能用作于int或float的操作，也同样能作用于BigInteger或BigDecimal）==，运算速度较慢，即以速度换取精度。BigInteger支持任意精度的整数，BigDecimal支持任意精度的定点数。

13\.Java确保数组会被初始化，而且不能在它的范围之外被访问。当创建一个数组对象时，实际上就是创建了一个引用数组，并且每一个引用都会自动被初始化为null。一旦Java看到null，就知道这个引用还没有被指向某个对象。在使用任何引用前，必须为其指定一个对象；如果试图使用一个还是null的引用，在运行时将会报错。
> Java还可以创建用来存放基本数据类型的数组。同样，编译器也能确保这种数组的初始化，因为它会将这种数组所占的内存全部置零。

14\.在C和C++中下述代码合法，但是在Java中却不能这样写：
```java
{
	int x = 12;
	{
		int x = 96;//illegal in Java，因为Java设计者认为这样做会导致程序混乱
	}
}
```
然而，Java中类变量和局部变量可以同名，如下所示：
```java
public class Test {
	int x = 12;
	public static void main(String[] args) {
		int x = 96;//legal in Java
	}
}
```

15\.对象的作用域：
```java
{
  String s = new String("a string");
  //end of scope
}
```
上述代码中，引用s在作用域终点就消失了。然而，s指向的String对象仍继续占据内存空间。虽然如此，但是在这段代码中，我们无法在这个作用域之外访问这个对象，因为对它唯一的引用已经超出了作用域的范围。在程序执行过程中，需要传递和复制对象引用，以便继续使用对象。是谁证明，由new创建的对象，只要你需要，就会一直保留下去。Java有一个**垃圾回收器**，用来监视用new创建的所有对象，并辨别那些不会再被引用的对象，之后释放这些对象的内存空间，以便供其他新的对象使用。

16\.类包括两种类型的元素：字段和方法。如果字段是对某个对象的引用，那么必须初始化该引用，以便使其与一个实际的对象（使用new来实现）相关联。若类的某个成员是基本数据类型，即使没有进行初始化，Java也会确保它获得一个默认值，如下表所示：

|基本类型|默认值|
|:---|:---:|
|boolean|false|
|char|'\u0000'(null)|
|byte|(byte)0|
|short|(short)0|
|int|0|
|long|0L|
|float|0.0f|
|fouble|0.0d|

注意，当变量是类的成员时，Java才确保给定其默认值，以确保那些基本类型的成员变量得到初始化，防止产生错误。然而，上述的初始化的方法并不适用于*局部变量*（即并非某个类的字段），因此在某个方法中定义有`int x ;`,那么变量x得到的可能是任意值（与`C`和`C++`中一样），而不会被自动初始化为零。若使用一个未初始化的局部变量，Java编译时会返回一个错误（而`C`和`C++`C++编译器中对此只是给予警告）。

17\.面向对象的程序设计通常简单地归纳为++“向对象发送消息”++，在`int x = a.f()`中消息是f()，对象是a。

18\.Java中传递对象，实际上传递的是引用。Java消除了所谓的“向前引用”问题，++即可以先使用类，后定义类++。

19\.static提供对象共享数据，引用static变量（方法）有两种方式，通过一个对象去定位它，也可通过其类名直接引用，而对非静态成员则必须通过对象去引用。static数据和static方法，亦可称为类数据和类方法。
> 一个static字段对每个类来说都只有一份存储空间，而非static字段则是对每个对象有一个存储空间，但是如果static用于某个方法，差别却没有那么大。

20\.javadoc只能为public和protected成员进行文档注释。所有javadoc命令都只能在`/**`注释中出现，结束于`*/`。使用javadoc的方式主要有两种：嵌入HTML或使用“文档标签”。

21\.独立文档标签是一些以“@”字符开头的命令，且要置于注释行的最前面（但是不算前导“*”之后的最前面）。而"行内文档标签"则可以出现在javadoc注释中的任何地方，它们也是以“@”字符开头，但是要括在花括号内。共有三种类型的注释文档（类注释、域注释和方法注释），分别对应于注释位置后面的三种元素：类、域和方法。

22\.一些可用于代码的javadoc标签：
==1）@see==
>@see:引用其他类的文档
>>example:
@see classname
@see fully-qualified-classname
@see fully-qualified-classname#method-name

注意：使用“@see”，文档中会出现“See Also”超链接文本

==2）@{@link package.class#member label}==
>该标签与@see极其类似，只是它用于行内，并且是用"label"作为超链接文本而不是“See Also”。

==3）@docRoot==
>该标签产生到文档根目录的相对路径，用于文档树页面的显示超链接。

==4）@inheritDoc==
>该标签从当前这个类的最直接的基类中继承相关文档到当前的文档注释中。

==5）@version==
>格式如下：
@version version-information
"version-information"可以是任何你认为适合包含在版本说明中的重要信息。
如果javadoc命令行使用了“-version”标记，那么就会从生成的HTML文档中特别提取出版本信息。


==6）@author==
>格式如下：
@author author-information
"author-information"是你的姓名，但是也可以包括电子邮件地址或其他任何适宜的信息。
如果javadoc命令行使用了“-author”标记，那么就会从生成的HTML文档中特别提取出作者信息。
可以使用多个标签以便列出所有作者，但是它们必须是连续放置。

==7）since==
>该标签允许你指定程序代码最早使用的版本，可以在HTML Java文档中看到它被用来指定所用的JDK版本的情况。

==8）@param==
>该标签用于方法文档，格式如下：
@param parameter-name description
其中，**parameter-name**是方法的参数列表中的标识符，**description**是可以延续数行的文本，终止于新的文档标签出现之前。可以使用任意多个这种标签，大约每一个参数都有一个这样的标签。

==9）@return==
>该标签用于方法文档，格式如下：
@return description
其中，**description**用来描述返回值的含义，可以延续数行。

==10）@throws==
>异常标签，格式如下：
@throws fully-qualified-calss-name description
其中，**fully-qualified-calss-name**给出一个异常类的无歧义的名字，而该异常类在别处定义。**description**告诉你为什么此特殊类型的异常会在方法调用中出现，可延续数行。

==11）@deprecated==
>该标签用于指出一些旧特性已由新特性所取代，建议用户不要再使用这些旧特性，因为在不久的将来它们可能会被删除。如果使用一个标记为@deprecated的方法，则会引起编译器发出警告。
>>在Java SE5中，Javadoc标签@deprecated已经被@Deprecated注解所替代。

23\.文档示例
```java
//:edu/ustc/home/HelloDate.java
package edu.ustc.home;

import java.util.*;

/**
 * The first Thinking in Java example program.
 * Displays a string and today's date.
 * @author yuchen
 * @author home.ustc.edu.cn/~wzzju
 * @version 4.0
 *
 */
public class HelloDate {
	/**
	 * Entry point to a class &  aplicatio.
	 * @param args array of string arguments
	 * @throws exceptions No exceptions thrown
	 */
	public static void main(String[] args) {
		System.out.println("Hello,it's: ");
		System.out.println(new Date());
	}/*Output:(55% match)
	Hello,it's:
	Tue Mar 01 11:04:30 CST 2016
	*///:~
}
```
注：第一行采用Thinking in Java作者的方法，用一个":"作为特殊记号说明这是包含源文件名的注释行。该行包含文件的路径信息，随后是文件名。最后一行的"///:~"标志源代码清单的结束。

24\.Java编程语言编码约定中，代码风格是这样规定的：类名的首字母要大写；如果类名由几个单词构成，那么把它们并在一起（也即是说，不要用下划线分隔名字），其中每个内部单词的首字母都采用大写形式。如：
```java
class AllTheColorsOfTheRainbow{//...
```
这种风格有时被称为“驼峰风格”。几乎其他所有内容——方法、字段（成员变量）以及对象引用名称等，公认的风格与类风格一样，只是标识符的第一个字母采用小写。如：
```java
class AllTheColorsOfTheRainbow{
	int anIntegerRepresentingColors;
	void changeTheHueOfTheColor(int newHue){
	//...
	}
	//...
}
```
当然，用户必须键入所有这些长名字，并且不能输错，因此一定与格外仔细。
