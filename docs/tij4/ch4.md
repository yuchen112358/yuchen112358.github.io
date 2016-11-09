##Java编程思想（第四版）学习笔记
####By 王震 <wzzju@mail.ustc.edu.cn>
***
###第四章 控制执行流程
#####1.关于goto
Java并不支持goto语句，但在Java中仍然可以进行类似goto那样的跳转，然而与典型的goto相比，存在很多限制。
#####2.再次注意
Java不允许我们将一个数字作为布尔型值使用！！  
在c/c++中的if(a)代码，于Java中必须写成if(a != 0)。
#####3.关于static
在一个类中，static函数只能访问static字段，并且只能直接调用类中的static方法。
如下：
```Java
public static void main(String[] args) {
    test(10, 5);
    print(result);
}
```
test方法和result字段必须是由static修饰的。
#####4.逗号操作符
Java里唯一用到逗号操作符的地方就是for循环的控制表达式。在控制表达式的初始化和步进部分，可以使用一系列由逗号分隔的语句；而且那些语句均会独立顺序执行。如：
```Java
for(int i = 1, j = i + 10/*初始化部分可以有任意数量的同一类型定义*/; i < 5; i++, j = i * 2) {
      System.out.println("i = " + i + " j = " + j);
}
```
#####5.Foreach语法
**示例1：**
```Java
public class ForEachFloat {
  public static void main(String[] args) {
    Random rand = new Random(47);
    float f[] = new float[10];
    for(int i = 0; i < 10; i++)
      f[i] = rand.nextFloat();
	//下面是foreach语法
    for(float x : f)
      System.out.println(x);
  }
}
```
任何返回一个数组的方法都可以使用foreach，如示例2。  
**示例2：**
```Java
public class ForEachString {
  public static void main(String[] args) {
    for(char c : "An African Swallow".toCharArray() )
      System.out.print(c + " ");
  }
}
```
#####6.return
如果在返回void的方法中没有return语句，那么在该方法的结尾处会有一个隐式的return。  
==注意==：如果一个方法声明它将返回void之外的其他东西，那么必须确保每一条代码路径都将返回一个值。
#####7.无穷循环
两种形式：while(true)和for(;;)，并且编译器将它们看作是同一回事。
#####8.goto
尽管goto仍然是Java中的一个保留字，但在语言中并未使用它；Java没有goto。然而,Java也能完成一些类似于跳转的操作，由break和continue这两个关键字实现，它们使用与goto相同的机制：标签。如：
```Java
label1:
```
在Java中，标签起作用的唯一的地方刚好是在迭代语句之前。“刚好”一词表明，在标签和迭代之间置入任何语句都不好，也都不允许。而在迭代之前设置标签的唯一理由是：我们希望在其中嵌套另一个迭代或者一个开关（switch）。这是由于break和continue关键词通常只中断当前循环，但若是随同标签一起使用，它们就会中断循环，直到标签所在的地方。下面详细说明：
```Java
label1:
	outer-iteration{
	  inner-iteration{
		  //...
		  break;//(1)
		  //...
		  continue;//(2)
		  //...
		  continue label1;//(3)
		  //...
		  break label1;//(4)		  
	  }
  }
```
在（1）中，break中断内部迭代，回到外部迭代。在（2）中，continue使执行点移回到内部迭代的起始处。在（3）中，continue label1同时中断内部迭代以及外部迭代，直接转到label1处；随后，它实际上是继续迭代过程，但却从外部迭代开始。在（4）中，break label1也会中断所有迭代，并回到labe1处，但并不重新进入迭代，即它实际是完全中止了两个迭代。
示例代码：
```Java
public class LabeledFor {
  public static void main(String[] args) {
    int i = 0;
    outer: // Can't have statements here
    for(; true ;) { // infinite loop
      inner: // Can't have statements here
      for(; i < 10; i++) {
        print("i = " + i);
        if(i == 2) {
          print("continue");
          continue;
        }
        if(i == 3) {
          print("break");
          i++; // Otherwise i never
               // gets incremented.
          break;
        }
        if(i == 7) {
          print("continue outer");
          i++; // Otherwise i never
               // gets incremented.
          continue outer;
        }
        if(i == 8) {
          print("break outer");
          break outer;
        }
        for(int k = 0; k < 5; k++) {
          if(k == 3) {
            print("continue inner");
            continue inner;
          }
        }
      }
    }
    // Can't break or continue to labels here
  }
} 
```
==注意==：如果不带标签，break和continue只能中断最内层的循环。
当然，如果想在中断循环的时候同时退出，简单的一个return即可。
#####9.关于break和continue用法的总结
break和continue可用在for循环和while循环中，规则如下：
1. 一般的continue会退回最内层循环的开头（顶部），并继续执行；
2. 带标签的continue会到达标签的位置，并重新进入紧接在那个标签后面的循环。
3. 一般的break会中断并跳出当前循环。
4. 带标签的break会中断并跳出标签所指的循环。  

==重点记忆==：
> 在Java中需要使用标签的唯一理由就是因为有循环嵌套存在，而且像从多层循环中break或continue。

#####10.switch
switch的选择因子现在不仅可以是int或char类型这样的整数值，也可以是字符串。但是依旧不能是浮点数（可以用enum减弱这种限制）。  
==Tips==:为将一个整数值（如int型）当作字符打印，必须将其转型为char；否则，将产生整型输出。

