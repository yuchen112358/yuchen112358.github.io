## 第七章 复用类
***
#### 1. 组合语法
1）对于非基本类型的对象，必须将其引用置于新的类中，但可以定义基本类型数据。  
2）每一个非基本类型的对象都有一个toString()方法，而且当编译器需要一个String而你却只有一个对象时，该方法便会被调用。  
3）初始化一个类中的对象引用有如下四种方式：
* 1.在定义对象的地方。这意味着它们总能够在构造器调用之前被初始化。
* 2.在类的构造器中。
* 3.就在正要使用这些对象之前，这种方式成为惰性初始化(Delayed initialization)。在生成对象不值得及不必每次都生成对象的情况下，这种方式可以减少额外的负担。
* 4.使用实例初始化。
以下是这四种方式的示例：

```Java
// Constructor initialization with composition.
import static net.mindview.util.Print.*;

class Soap {
  private String s;
  Soap() {
    print("Soap()");
	//2.在类的构造器中初始化
    s = "Constructed";
  }
  public String toString() { return s; }
}	

public class Bath {
  private String // 1.在定义对象的地方初始化：
    s1 = "Happy",
    s2 = "Happy",
    s3, s4;
  private Soap castille;
  private int i;
  private float toy;
  public Bath() {
    print("Inside Bath()");
	//2.在类的构造器中初始化：
    s3 = "Joy";
    toy = 3.14f;
    castille = new Soap();
  }	
  // 4.实例初始化:
  { i = 47; }
  public String toString() {
    if(s4 == null) // 3.惰性初始化(Delayed initialization):
      s4 = "Joy";
    return
      "s1 = " + s1 + "\n" +
      "s2 = " + s2 + "\n" +
      "s3 = " + s3 + "\n" +
      "s4 = " + s4 + "\n" +
      "i = " + i + "\n" +
      "toy = " + toy + "\n" +
      "castille = " + castille;
  }	
  public static void main(String[] args) {
    Bath b = new Bath();
    print(b);
  }
} /* Output:
Inside Bath()
Soap()
s1 = Happy
s2 = Happy
s3 = Joy
s4 = Joy
i = 47
toy = 3.14
castille = Constructed
*///:~
```
#### 2.继承语法
1）当创建一个类时，总是在继承，因此，除非已明确指出要从其他类中继承，否则就是在隐式地从Java的标准根类Object进行继承。  

2）继承关键字`extends`：继承会自动得到基类中所有的域和方法。  

3）调用另外一个类的main函数的方式与调用另外一个类的普通静态函数相同，即`类名.main(args);`，args可以是主调用类从命令行获得的参数，也可以是其他任意的String数组。  

4）可以为每个类都创建一个main方法。这种在每个类中都设置一个main方法的技术可使每个类的单元测试都变得简单易行。而且在完成单元测试之后，也无需删除main(),可以留待下次测试。  

5）即使一个类只具有包访问权限，其public main()仍然是可以访问的。  

6）为了继承，一般的规则是将所有的数据成员都指定为private，将所有的方法指定为public（protected members also allow access by derived classes）。   

7）Java用==super==关键字表示超类（父类）。表达式`super.fun();`可以调用父类中的函数(此处是调用函数fun())。  

###### 2.1 初始化基类
==注意：==基类=父类；导出类=子类。  
1）当创建了一个导出类的对象时，该对象包含了一个基类的子对象，该子对象被包装在导出类对象内部。
2）基类子对象的初始化：在构造器中调用基类构造器来执行初始化。在执行基类构造器之前，定义处初始化、实例初始化等均会被执行。==Java会自动在导出类的构造器中插入对基类构造器的调用。==
示例代码：
```Java
// Constructor calls during inheritance.
import static io.github.wzzju.util.Print.*;

class Art {
  private String art = " test art.\n";
  private String artS;
  {
	  artS = " ART";
  }
  Art() { print("Art constructor"+art+artS); }
}

class Drawing extends Art {
  private String draw = " test drawing.\n";
  private String drawS;
  {
	  drawS = " DRAW";
  }
  Drawing() { print("Drawing constructor"+draw+drawS); }
}

public class Cartoon extends Drawing {
  public Cartoon() { print("Cartoon constructor"); }
  public static void main(String[] args) {
    Cartoon x = new Cartoon();
  }
} /* Output:
Art constructor test art.
 ART
Drawing constructor test drawing.
 DRAW
Cartoon constructor
*///:~
```
可以看出，构建过程是从基类“向外”扩散的，所以基类在导出类构造器可以访问它之前，就已经完成初始化了。当然，默认构造器也会逐层调度基类的构造器。
###### 2.2 带参数的构造器
* 编译器可以自动调用默认的构造函数，是因为它们没有任何参数。但是如果没有默认的基类构造函数，或者想调用一个带参数的基类构造函数，必须使用关键字super显示地编写调用基类构造函数的语句，并且配以适当的参数列表。
* 如果基类没有默认构造器（无参构造器），导出类不显式的调用基类的带参构造器，则编译器会报错。
* ==格外注意：==
	- 调用基类构造器必须是在导出类构造器中要做的第一件事，否则编译器会报错。
	- 而覆盖父类的方法时，可以在做完子类要做的事情之后再调用父类对应的方法。Java编程思想中在该种情况下都是放在最后才调用父类对应的方法。

#### 3.代理
代理是第三种复用代码的关系，Java并没有提供对它的直接支持。它是继承和组合之间的中庸之道：
* 首先，我们需要将一个成员对象置于所要构造的类中（就像组合）；
* 其次，我们需要在新类中暴露该成员对象的所有方法（就像继承）或该成员对象的所有方法的某个子集。  

示例代码：

```Java
public class SpaceShipControls {
  void up(int velocity) {}
  void down(int velocity) {}
  void left(int velocity) {}
  void right(int velocity) {}
  void forward(int velocity) {}
  void back(int velocity) {}
  void turboBoost() {}
}

public class SpaceShipDelegation {
  private String name;
  private SpaceShipControls controls =
    new SpaceShipControls();
  public SpaceShipDelegation(String name) {
    this.name = name;
  }
  // Delegated methods:
  public void back(int velocity) {
    controls.back(velocity);
  }
  public void down(int velocity) {
    controls.down(velocity);
  }
  public void forward(int velocity) {
    controls.forward(velocity);
  }
  public void left(int velocity) {
    controls.left(velocity);
  }
  public void right(int velocity) {
    controls.right(velocity);
  }
  public void turboBoost() {
    controls.turboBoost();
  }
  public void up(int velocity) {
    controls.up(velocity);
  }
  public static void main(String[] args) {
    SpaceShipDelegation protector =
      new SpaceShipDelegation("NSEA Protector");
    protector.forward(100);
  }
}
```
==使用代理的好处：==使用代理时，我们可以拥有更多的控制力，因为我们可以选择只提供在成员对象中的方法的某个子集。

#### 4.结合使用组合和继承
同时使用组合和继承，并配以必要的构造器初始化，可以创建更加复杂的类。
###### 4.1.确保正确清理
```Java
try{
	//......
}finally{
	x.dispose();
}
```
上述代码中的finally子句表示的是“无论发生什么事，一定要为x调用dispose()。”
在清理方法（dispose()）中，必须注意对基类清理方法和成员对象清理方法的调用顺序，以防某个子对象依赖于另外一个子对象的情形发生。  
* 一般，采用与C++编译器在其析构函数上所施加的形式：==首先，执行类的所有特定的清理工作，其顺序同生成顺序相反（通常这就要求基类元素仍旧存活）；然后调用基类的清理方法。==
* ==注意：==除了内存以外，不能依赖垃圾回收器去做任何事。如果需要进行清理，最好编写自己的清理方法，但是不要使用finalize()。

###### 4.2名称屏蔽
* 如果Java的基类拥有某个已被多次重载的方法名称，那么在导出类中重新定义该方法名称并不会屏蔽器在基类中的任何一个版本（这一点与C++不同）。因此，无论是在该层或者它的基类中对方法进行定义，重载机制都可以正常工作。
* 如果你只是想覆写某个方法，但是害怕不留心重载了该方法（而并非覆写了该方法）时，可以选择添加`@Override`注解（Java SE5新增）。
* 在一个方法前添加了`@Override`注解，该方法便只能是覆写父类的某个方法，若是不留心写成了重载，编译器便会报错。这样`@Override`注解便可以防止你在不想重载时而意外地进行了重载。

#### 5.在组合和继承之间选择
* 组合技术通常用于想在新类中使用现有类的功能而非它的接口这种形式。即，在新类中嵌入某个对象，让其实现所需要的功能，但是新类的用户看到的只是为新类所定义的接口，而非所嵌入对象的接口。为取得此效果，需要在新类中嵌入一个现有类的private对象。（特例，如Car将成员对象（Window/Wheel/Engine...）声明为public，一般情况下应该使域成为private。）
* 在继承的时候，使用某个现有类，并开发一个它的特殊版本。通常，这意味着你在使用一个通用类，并为了某种特殊需要而将其特殊化。
* “is-a”(是一个)的关系是用继承来表达的，而“has-a”(有一个)的关系则是用组合来表达的。

#### 6.protected关键字
* ==protected关键字的作用：==就类用户而言，其是private的，但对于任何继承于此类的子类或其他任何位于同一个包内的类来说，它却是可以访问的。（protected也提供了包内访问权限。）  
* ==注意：==尽管可以创建protected域，但是最好的方式还是将域保持为private；你应当一直保留“更改底层实现”的权利。然后通过protected方法来控制类的继承者的访问权限。

```Java
class A{
  private String name;
  protected void set(String name){
    this.name = name;
  }
  public A(String nm){
    name = nm;
  }
  //......
}

class B extends A{
  private int number;
  public B(String nm, int num){
    super(nm);
	number = num;
  }
  public void change(String nm, int num){
    set(nm);//因为set()函数是protected的，故无论B和A是否在一个包内，在此都可以访问。
	number = num;
  }
  //......
}
```
#### 7.向上转型
<div align = "center">
![类继承图](../../images/posts/tij_reusing00.png)  
类继承图
</div>

由导出类转型成基类，在继承涂上是向上移动的，因此一般称为*向上转型(upcasting)*。由于向上转型是从一个较专用类型向较通用类型转换，所以总是很安全的。换句话说，导出类是基类的一个超集。它可能比基类含有更多的方法，但它必须至少具备基类中所含有的方法。在向上转型的过程中，类接口中唯一可能发生的事情是丢失方法，而不是获取它们。
###### 再讨论组合和继承
相比于继承，运用组合技术使用现有类来开发新的类的做法更常见。应该慎用继承技术。
* 到底是该用组合还是用继承，一个最清晰的判断方法就是问一问自己是否需要从新类向基类进行向上转型。如果必须向上转型，则继承是必要的；但是如果不需要，则应当好好考虑自己是否需要继承。

#### 8.final关键字
使用final的含义是"这是不可改变的"，不想做改变可能出于两种理由：设计或效率。
final可用作数据、方法和类的修饰。
###### 8.1 fianl 数据
* 恒定不变的数据有两种：  
	1\. 一个永不改变的编译时常量。
	* 例子：
```Java
private final int valueOne = 9;
private static final int VALUE_TWO = 99;
//Typical public constant
public static final int VALUE_THREE = 89;
```

	2\. 一个在运行时被初始化的值，而且不希望被改变。
	* 例子：
```Java
private static Random rand = new Random(47);
private final int i4 = rand.nextInt(20);
static final int INT_5 = rand.nextInt(20);
```

* 一个既是static又是final的域只占据一段不能改变的存储空间。注意，带有恒定初始值(编译时常量)的static final基本类型全用大写字母命名，且字与字之间用下划线隔开。
* 对于基本类型，final使数值不变；而用于对象引用，final使引用恒定不变，即，一旦引用被初始化指向一个对象引用，就无法再把它指向另一个对象。然而，对象其自身却是可以修改的（如对象内部的某个字段的值是可以被改变的）。该规则同样适用于数组，数组只不过是另一种引用。

###### 8.1.1 空白final
空白final是指被声明为final但又未给定初值的域。但空白final必须在构造器中用表达式赋值。例子如下：
```Java
class Poppet {
  private int i;
  Poppet(int ii) { i = ii; }
}

public class BlankFinal {
  private final int i = 0; // Initialized final
  private final int j; // Blank final
  private final Poppet p; // Blank final reference
  // Blank finals MUST be initialized in the constructor:
  public BlankFinal() {
    j = 1; // Initialize blank final
    p = new Poppet(1); // Initialize blank final reference
  }
  public BlankFinal(int x) {
    j = x; // Initialize blank final
    p = new Poppet(x); // Initialize blank final reference
  }
  public static void main(String[] args) {
    new BlankFinal();
    new BlankFinal(47);
  }
}
```
总之，必须在域的定义外或每个构造器中用表达式对fianl进行赋值。

###### 8.1.2 fianl参数
Java允许在参数列表中以声明的方式将参数指明为final。这意味着你无法在方法中更改参数引用**所指向的对象**(但对象本身却是可以改变的)或无法在方法中更改基本类型的值。该特性注意用来向匿名内部类传递数据。
```Java
void with(final Gizmo g) {
	  g.i++;//OK
    //! g = new Gizmo(); // Illegal -- g is final
  }
  void without(Gizmo g) {
    g = new Gizmo(); // OK -- g not final
    g.spin();
  }
  // void f(final int i) { i++; } // Can't change
  // You can only read from a final primitive:
  int g(final int i) { return i + 1; }
```
###### 8.2 final方法
* 使用final方法的原因有两个：
	* 把方法锁定，以防任何继承类修改它的含义。这是出于设计的考虑：确保在继承类中使方法行为保持不变，并且不会被覆盖。
	* 效率：类似于C++的inline机制，早期的虚拟机需要，现在不需要了，故现在不需要使用final方法进行优化了。
综上，仅当你想显式地阻止覆盖该方法时，才使该方法成为final的。

* final和private关键字
	* 类中所有的private方法都隐式地指定为是final的。可以对private方法添加final修饰词，但这并不能给该方法增加任何额外的意义。

看下面一个特殊的例子：

```Java
class WithFinals {
  // Identical to "private" alone:
  private final void f() { print("WithFinals.f()"); }
  // Also automatically "final":
  private void g() { print("WithFinals.g()"); }
}

class OverridingPrivate extends WithFinals {
  private final void f() {
    print("OverridingPrivate.f()");
  }
  private void g() {
    print("OverridingPrivate.g()");
  }
}

class OverridingPrivate2 extends OverridingPrivate {
  public final void f() {
    print("OverridingPrivate2.f()");
  }
  public void g() {
    print("OverridingPrivate2.g()");
  }
}

public class FinalOverridingIllusion {
  public static void main(String[] args) {
    OverridingPrivate2 op2 = new OverridingPrivate2();
    op2.f();
    op2.g();
    // You can upcast:
    OverridingPrivate op = op2;
    // But you can't call the methods:
    //! op.f();
    //! op.g();
    // Same here:
    WithFinals wf = op2;
    //! wf.f();
    //! wf.g();
  }
} /* Output:
OverridingPrivate2.f()
OverridingPrivate2.g()
*///:~
```
上例中，我们试图覆盖一个private方法(隐含是final的)，似乎是奏效的，而且编译器也不给出错误信息。但是上面的代码并没有进行对基类方法的覆写，仅是生成了一个新的方法，所以不可运用多态的性质。  
==注意： "覆盖"只有在某方法是基类的接口（public方法）的一部分时才会出现。即，必须能将一个对象向上转型为它的基类并可调用相同的方法。==

###### 8.3 final类
final置于类的定义之前表示==该类不允许被继承==。这样做的原因如下：
* 出于某种考虑，你对该类的设计永不需要做任何变动；
* 出于安全的考虑，你不希望它有子类。

关于final类：
* final类的域可以根据个人意愿选择为是或不是final。不论类是否被定义无final，相同的规则都适用于定义为final的域，如final类的非final域是可以被重新赋值的；
* 由于final类禁止继承，所以final类中所有的方法都隐式指定为final的，因为本来就无法覆盖它们。当然，在final类中可以给方法添加final修饰，但这不会添加任何额外的意义。

现代Java的容器库用ArrayList替代了Vector，用HashMap代替了HashTable。

#### 9.初始化及类的加载
* 每个类的编译代码都存在于它自己的独立的文件中。该文件只有在需要使用程序代码时才会被加载。
* 类代码在初次使用时才加载。即，类加载发生于创建类的第一个对象之时，但是当访问static域或static方法时，也会发生加载。
* 初次使用之处也是static初始化发生之处。所有的static对象和static代码段都会在加载时依程序中的顺序(即，定义类时的书写顺序)而依次初始化。当然，定义为static的东西只会被初始化一次。

类的构造器也是static方法（即使static关键字没有显式地写出来）。因此更准确地讲，类是在其他任何static成员被访问时加载的。

##### 初始化及类的加载过程([点击查看](/resources/class_load_process.pdf))  

* 示例代码如下：

```Java
import static io.github.wzzju.util.Print.*;

class Insect {
  private int i = 9;
  protected int j;
  Insect() {
    print("i = " + i + ", j = " + j);
    j = 39;
  }
  private static int x1 =
    printInit("static Insect.x1 initialized");
  static int printInit(String s) {
    print(s);
    return 47;
  }
}

public class Beetle extends Insect {
  private int k = printInit("Beetle.k initialized");
  public Beetle() {
    print("k = " + k);
    print("j = " + j);
  }
  private static int x2 =
    printInit("static Beetle.x2 initialized");
  public static void main(String[] args) {
    print("Beetle constructor");
    Beetle b = new Beetle();
  }
} /* Output:
static Insect.x1 initialized
static Beetle.x2 initialized
Beetle constructor
i = 9, j = 0
Beetle.k initialized
k = 47
j = 39
*///:~
```









































