##Java编程思想（第四版）学习笔记
####By 王震 <wzzju@mail.ustc.edu.cn>
***
###第三章 操作符
#####1\.静态导入（static import)  
格式如下：  
**import static com.**...**.ClassName.\*** ;

此处多了个static关键字，并且在类名ClassName后面多了个 .\* ，意思是导入这个类里的所有静态方法和静态字段（属性）。当然，也可以只导入某个静态方法，只要把 .\* 换成静态方法名就行了。然后在这个类中，就可以直接用方法名调用静态方法，而不必用ClassName.方法名的方式来调用。这种方法的好处就是可以简化一些操作，例如打印操作System.out.println(...);就可以将其写入一个静态方法print(...)，在使用时直接print(...)就可以了。 但是这种方法建议在有很多重复调用的时候使用，如果仅有一到两次调用，不如直接写来的方便。

示例如下：
```Java
//: net/mindview/util/Print.java
// Print methods that can be used without
// qualifiers, using Java SE5 static imports:
package edu.ustc.home.util;
import java.io.*;

public class Print {
  // Print with a newline:
  public static void print(Object obj) {
    System.out.println(obj);
  }
  // Print a newline by itself:
  public static void print() {
    System.out.println();
  }
  // Print with no line break:
  public static void printnb(Object obj) {
    System.out.print(obj);
  }
  // The new Java SE5 printf() (from C):
  public static PrintStream
  printf(String format, Object... args) {
    return System.out.printf(format, args);
  }
} ///:~

```
使用上述代码：
```Java
import static net.mindview.util.Print.*;

public class A {
  public static void main(String[] args) {
    print("Hello,static import!");
  }
}
```

#####2\.Random 类的使用：
>1.	要生成随机数，可以先创建一个Random类的对象，若在创建的过程中没有传递任何参数，那么Java就会将当前时间作为随机数生成器的种子，并由此在程序每次执行时都会产生不同的随机数；  
2.	若想让程序每次执行时都产生相同的随机数（以使得输出可验证），可通过在创建Random对象时提供种子（用于数生成器的初始化值，随机数生成器对于特定的种子值总是产生相同的随机数序列）。
示例：
```Java
Random rand = new Random(47);//每次运行结果相同
Random randDif = new Random();//每次运行结果不同
int i;
// Choose value from 1 to 100:
//rand.nextInt(100)设置了所产随机数的上限为100，而其下限为0，为避免除零，应该进行加1操作。
i = rand.nextInt(100) + 1;
float u; // Applies to doubles, too
//chosen (approximately) uniformly from the range 0.0f (inclusive) to 1.0f (exclusive)
u = rand.nextFloat();
long l;
l = rand.nextLong();
double d;
//chosen (approximately) uniformly from the range 0.0d (inclusive) to 1.0d (exclusive)
d = rand.nextDouble();
```

#####3\.一元减号用于转变数据的符号，而一元加号只是为了与一元减号相对应，但是它唯一的作用仅仅是将较小数据类型的操作数提升为int。
#####4\.++和--说明
对于前缀形式（\++i和--i），在执行完运算后才得到值；对于后缀形式(i\++和i--)，则是在执行运算之前就得到值。
#####5\.关系操作符
关系操作符中，等于和不等于适用于所有基本数据类型，而其他比较符不适用于boolean类型，因为boolean值只能为true或false，“大于”和“小于”没有实际意义。
>==和!=也适用于所有对象,但是它们比较的是对象的引用。要比较两个对象的实际内容，应该使用所有对象都适用的特殊方法==equals()==。但是该方法不适用于基本类型，基本类型直接使用==和!=即可。
>>注意实现自己的类时，若想使得该类的两个对象的内容可以作比较运算，应该覆盖自己设计类中的equals()方法，因为equals()的默认行为是比较对象的引用。

######6\.逻辑操作符
"&&"、“||”、“!”操作只可应用于布尔值，与C和C++不同的是：不可以将一个非布尔值当做布尔值在逻辑表达式中使用。
######7\.关于二进制
在C、C++或Java中，二进制数没有直接常量表示方法。但是，通过Integer和Long的静态方法toBinaryString()可以很容易地实现二进制形式输出。
######8\.指数计数法
==“e”==代表10的幂次，如1.39e-43f表示1.39x10^-43^，编译器将指数字面量当成double处理，所以在指数常量后最好加上后缀"d"和"f"。
######9\.按位操作符(只可用来操作整数基本数据类型中的单个“比特位”)
`&、|、^、~`分别为按位“与”、按位“或”、按位“异或”和按位“非”，“~”是一元操作符，所以不可与“=”联合使用，其他三个可以。
>我们可将布尔类型作为一种单比特值对待，并且可对布尔类型执行按位“与”、按位“或”和按位“异或”运算，但不能执行按位“非”。对于布尔值，按位操作符具有与逻辑操作符相同的效果，只是它们不会中途“短路”。此外，针对布尔值进行的按位运算为我们新增了一个“异或”逻辑操作符，它并未包括在“逻辑”操作符的列表中。

#####10\.移位操作符（只可用来处理整数基本数据类型中的二进制“位”）
>1. `<<`，左移位操作符，低位补零；  
>2. `>>`，右移位操作符，“有符号”右移位操作符使用“符号扩展”：若符号为正，则高位补零；若符号为负，则高位补1；  
>3. `>>>`，“无符号”右移位操作符，使用“零扩展”：无论正负，都在高位补零。这一操作符是C或C++中所没有的。  

#####11.三元操作符if-else
```Java
boolean-exp ? value0 : value1
```
#####12.字符串操作符+和+=（字符串连接符）
```Java
package io.github.wzzju; /* Added by Eclipse.py */
import static io.github.wzzju.util.Print.*;
public class StringOperators {
  public static void main(String[] args) {
    int x = 0, y = 1, z = 2;
    String s = "x, y, z ";
    print(s + x + y + z);//输出结果是x, y, z 012
	print(x + y + z + s);//输出结果是3x, y, z 
    print(x + " " + s); // Converts x to a String
    s += "(summed) = "; // Concatenation operator
    print(s + (x + y + z));
    print("" + x); // Shorthand for Integer.toString()
  }
}
```

#####13.使用操作符常犯的错误
```Java
while(x = y){
//...
}
```
上述代码中只要x,y不都为布尔值，在Java中就不会编译通过，因为Java不会自动将int数值转换为布尔值。Java编译器不会允许我们随便把一种类型当做另外一种类型来用。

#####14.关于类型转换
* 窄化转换（narrowing conversion），必须显示进行类型转换。
* 扩展转换（widening conversion），不必显示进行类型转换。

==注意:==Java允许我们把任何基本数据类型转换为别的基本数据类型，但布尔型除外，因为布尔型根本不允许进行任何类型的转换处理。  
将float或double转型为整型值时，总是对该数字执行结尾。如果想要得到四舍五入的结果，就需要使用java.lang.Math中的round()方法，由于round()是java.lang的一部分，因此使用它时不需要额外导入。  
对基本数据类型执行算术运算或按位运算，只要类型比int小（即char、byte或者short），在运算之前这些值会自动转换成int。  
######15.Java没有sizeof
sizeof在c/c++中存在主要是为了解决移植问题的，但是在Java中所有数据类型在所有机器中的大小都是相同的，所有不需要sizeof()操作符来解决移植问题。
#####16.关于布尔型值
能够对布尔型值进行的运算非常有限。我们只能赋予true和false值，并测试它的真假，而不腻对布尔值进行其他任何运算（如加、减等）。
#####17.关于溢出
两个较大的int值相乘可能会溢出，但是编译器不会给出任何出错或警告的信息。
