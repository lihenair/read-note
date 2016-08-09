##泛型设计
###WHY
Generic programming
泛型提供了一个解决方案：**类型参数(type paramters)**。编译器可以利用类型参数进行检查，避免在类运行时强制类型转换发生的异常。
类型参数的魅力：程序的可读性和安全性。

泛型程序设计计划分为3个能力级别：基本级别是，仅仅使用泛型类——典型的是像ArrayList这样的集合——不必考虑透明的工作方式和原因。

###简单定义泛型类

```
public class Pair<T> {
    private T first;
    private T second;
    
    public Pair() {
        first = null;
        second = null;
    }
    
    public Pair(T first, T second) {
        this.first = first;
        this.second = second;
    }
    
    public T getFirst() { return first; }
    public T getSecond() { return second; }
    
    public void setFirst(T first) { this.first = first; }
    public void setSecond(T second) { this.second = second; }
```

一个**泛型类**就是具有一个或多个类型变量的类。
Pair类引入了一个类型变量T，用尖括号(<>)括起来，并放到类名的后面。泛型类可以有多个从类型参数。例如，可以定义Pair类，其中第一个域和第二个域使用不同的类型：```public class Pair<T, U> {}```。

> **注释**：类型变量使用大写形式，且比较短，这是很常见的。在Java库中，使用变量E表示集合的元素类型，K和V分别表示映射的关键字和值得类型。T(需要时还可以用邻近的字母U和S)表示“任意类型”。

泛型类可看做普通类的工厂。

###泛型方法

```
class ArrayAlg {
    public static <T> T getMiddle(T... a) {
        return a[a.length /2];
    }
}
```
这个方法是在普通类中定义的，而不是在泛型类中定义的。这是一个泛型方法，类型变量放到修饰符后面，尖括号里，返回类型前面。
泛型方法可以定义在普通类中，也可以定义在泛型类中。
当调用一个泛型方法时，在方法名前的尖括号中放入具体的类型：
```String middle = ArrayAlg.<String>getMiddle("Jonh", "Q", "Public");```。这种情况下，方法调用中可以省略<String>类型参数。编译器有足够的信息能够推断出所调用的方法。它用names的类型(即String[])与泛型类型T[]进行匹配并推断出T一定是String。也就是说可以调用```String middle = ArrayAlg.getMiddle(3.14, 1729, 0);```。

###类型变量的限定
有时，类或方法需要对类型变量加以约束。

```
class ArrayAlg {
    public static <T extends Comparable> T min(T[] a) {
        if (a == null || a.length == 0) return null;
        T smallest = a[0];
        for (T e : a) {
            if (smallest.compareTo(e) > 0) smallest = e;
        }
        return smallest;
    }
}
```
解决方案是将T限制为实现了Comparable接口的类。可以通过对类型变量T设置**限定(bound)**来实现：

```
public static <T extends Comparable> T min(T[] a)...
```
为什么使用关键字extends而不是implements？毕竟Comparable是一个接口。下面的符号：

```
<T extends BoundingType>
```
表示T应该是绑定类型的**子类型(subtype)**。T和绑定类型可以是类，也可以是接口。选择关键字extends的原因是更接近子类的概念，并且也不需要添加新的关键字。

一个类型变量或通配符可以有多个限定，例如

```
T extends Comparable & Serializable
```
限定类型用```&```分隔，而逗号用来分隔类型变量。

###泛型代码和虚拟机
虚拟机没有泛型类型对象————所有对象都属于普通类。
无论何时定义一个泛型类型，都自动提供了一个相应的**原始类型(raw type)**。原始类型的名字就是删去类型参数后的泛型类型名。**擦除(erased)**类型变量，并替换为限定类型(无限定的变量用Object)。

```
public class Pair {
    private Object first;
    private Object last;
    
    public Pair(Object first, Object last) {
        this.first = first;
        this.last = last;
    }
    
    public Object getFirst() { return first; }
    public Object getLast() { return last; }
    
    public void setFirst(Object first) { this.first = first; }
    public void setLast(Object last) { this.last = last; }
}
```
由于T是一个无限定的变量，所以直接用Object替换。
结果是一个普通的类，就像反省引入Java语言之前已经实现的那样。
原始类型用第一个限定的类型变量来替换，如果没有给定限定就用Object替换。

```
public class Interval<T extends Comparable & Serializable> implements Serializable {
    private T lower;
    private T upper;
    ...
    public Interval(T first, T second) {
        if (first.compareTo(second) <= 0) { lower = first; upper = second; }
        else { upper = first; lower = second; }
	}
}
```
原始类型Interval如下所示：

```
public class Interval implements Serializable {
    private Comparable lower;
    private Comparable upper;
    ...
    public Interval(Comparable first, Comparable second) {
        if (first.compareTo(second) <= 0) { lower = first; upper = second; }
        else { upper = first; lower = second; }
	}
}
```
> **注释：**读者可能想要知道切换限定:```class Interval<Serializable & Comparable>```会发生什么。如果这样做，原始类型用Serializable替换T，而编译器在必要时要向Comparable插入强制类型转换。为了提高效率，应该将标签(tagging)接口(即没有方法的接口)放到边界列表的末尾。

####翻译泛型表达式
当程序调用泛型方法时，如果擦除返回类型，编译器插入强制类型转换。例如，下面这个语句序列：

```
Pair<Employee> buddies = ...
Employee buddy = buddies.getFirst();
```
擦除getFirst的返回类型后，将返回Object类型。编译器自动插入Employee的强制类型转换。也就是说，编译器把这个方法调用翻译为两条虚拟机指令：

- 对原始方法Pair.getFirst的调用。
- 将返回的Object类型强制转换为Employee类型。

当存取一个泛型域时也要插入强制类型转换。```Employee buddy = buddies.first;```也会在结果字节码中插入强制类型转换。

####翻译泛型方法
类型擦除也会出现在泛型方法中。通常认为泛型

```
public static <T extends Comparable> T min(T[] a)
```
是一个完整的方法族，而擦除类型之后，只剩下一个方法：

```
public static Comparable min(Comparable[] a)
```
注意类型参数T已经被擦除了，只留下了限定类型Comparable。

需要记住有关Java泛型转换的事实：

- 虚拟机中没有泛型，只有普通的类和方法
- 所有的类型参数都用它们的限定类型替换
- 桥方法被合成来保持多态
- 为保持类型安全性，不要是插入强制类型转换

####调用遗留代码
使用**注解**消除警告。注解必须放在生成这个警告的代码所在的方法之前，如下：

```
@SuppressWarnings("unchecked")
Dictionary<Integer, Components> labelTable = slider.getLabelTable();
```
这个注解会关闭对方法中所有代码的检查。

###约束与局限性
####不能用基本类型实例化参数类型
不能用类型参数代替基本类型。没有Pair<double>，只有Pair<Double>，原因是类型擦除。擦除后，Pair类含有Object类型的域，而Object不能存储double值。这样做与Java语言中基本类型的独立状态一直。

####运行时类型查询只适用于原始类型
虚拟机中的对象总有有个特定的非泛型类型。因此，所有的类型查询只产生原始类型。

####不能创建参数化类型的数组
不能实例化参数化类型的数组，即不能new泛型数组。但是可以声明泛型数组。
> **注释：**可以声明统配类型的数组，然后进行类型转换。
> ```Pair<String>[] table = (Pair<String>) new Pair<?>[10];```
> 结果将是不安全的。

> **提示：**如果需要收集参数化类型对象，只有一种安全而有效的方法：使用ArrayList：ArrayList<Pair<String>>。

####Varargs警告

```
public static <T> void addAll(Collection<T> coll, T... ts) {
    for (t : ts) coll.add(t);
}

Collection<Pair<String>> table = ...;
Pair<String> pair1 = ...;
Pair<String> pair2 = ...;
addAll(table, pair1, pair2);
```
为了调用这个方法，JVM必须建立一个Pair<String>数组，这就违反了前面的规则。
可以使用@SuppressWarnings("unchecked")或者@SafeVarargs解决警告。

####不能实例化类型变量
不能使用像```new T()```，```new T[]```或者```T.class```这样的表达式中的类型变量。类型擦除将T改变成Object，而且本已肯定不是调用```new Object()```。但是可以通过反射调用Class.newInstance方法来构造泛型类型。
实现方式如下：

```
public static <T> Pair<T> makePair(Class<T> cl) {
    try { return new Pair<>(cl.newInstance(), cl.newInstance()); }
    catch (Exception e) { return null; }
}
```
这个方法可以按照下列方式调用：

```
Pair<String> p = Pair.makePair(String.class);
```
不能构造一个泛型数组：

```
public static <T extends Comparable> T[] minmax(T[] a) { T[] mm = new T[2]; }
```
类型擦除会让这个方法永远构造Object[2]数组。

可以利用反射，调用Array.newInstance：

```
public static <T extends Comparable> T[] minmax(T... a) {
    T[] mm = (T[]) Array.newInstance(a.getClass.getComponentType(), 2);
}
```

####反省类的静态上下文中类型变量无效
不能再静态域或方法中引用类型变量：

```
public class Singleton<T> {
    private static T singleInstance; // ERROR
    
    public static T getInstance() { // ERROR
        if (singleInstance == null) {
            singleInstance = new T();
        }
        return singleInstance;
    }
}
```
类型擦除后，只剩下Singleton类，它只包含一个singleInstance域。因此禁止使用带有类型变量的静态域和方法。

####不能抛出或捕获泛型类的实例
即不能抛出也不能捕获泛型类对象。实际上，甚至泛型类扩展Thrwoable都是不合法的。

```
public class Problem<T> extends Exception {/*...*/} // ERROR--can't extend Throwable
```
catch子句中不能使用类型变量。

```
public static <T extends Throwable> void doWork(Class<T> t) {
    try {
        do work
    } catch (T e) { // ERROR--can't catch tye variable
        Logger.global.info()
    }
}
```
异常范围中使用类型变量是允许的。

```
public static <T extends Throwable> void doWork(T t) {
    try {
        do work
    } catch (Throwable realCause) {
        Logger.global.info()
        t.initCause(realCause);
        throw t;
    }
}
```
通过使用泛型类、擦除和@SuppressWarnings注解，就能消除Java类型系统的部分基本限制。

####注意擦除后的冲突
当泛型类型被擦除时，无法创建引发冲突的条件。下面是一个实示例。假定像下面这样将equals方法添加到Pair类中：

```
public class Pair<T> {
    public boolean equals(T value) { return first.equals(value) && second.equals(value); }
}
```
考虑一个Pair<String>。从概念上讲，它有两个equals方法：

```
boolean equals(String) // define in Pair<T>
boolean equals(Object) // inherited from Object
```
方法擦除后就剩下boolean equals(Object)了，与Object.equals方法发生冲突。解决办法是重新命名引发错误的方法。

泛型规范说明还提到另外一个原则：“要想支持擦除的转换，就需要强行限制一个类或类型变量不能同时成为两个接口类型的子类，而这两个接口是同一个接口的不同参数化。”例如，下面的代码是非法的：

```
class Calendar implements Comparable<Calendar> {}
class GregorianCalendar extends Calendar implements Comparable<GregorianCalendar> {} // ERROR
```
其原因有可能与合成的桥方法产生冲突。实现了Comaprable<X>的类可以获得一个桥方法：

```
public int compareTo(Object other) { return compareTo(X) other; }
```
对于不同类型的X不能有两个这样的方法。

###泛型类型的继承规则
Pair\<Manager>是Pair\<Employee>的一个子类吗？答案是“不是”。无论S与T是什么联系，通常Pair\<S>与Pair\<T>没有什么联系。这一限制看起来过于严格，但对于类型安全非常必要。

永远可以将参数化类型转换为一个原始类型。转换成员是类型之后会产生类型错误。

泛型类可以货站或实现其他的泛型。例如，ArrayList\<T>类实现List\<T>接口。

###通配符类型
```
Pair<? extends Employee>
```
表示任何泛型Pair类型，它的类型参数是Employee的子类。

假设要编写一个打印雇员对的方法，像这样：

```
public static void printBuddies(Pair<Employee> p) {
    Employee first = p.getFirst();
    Employee last = p.getLast();
    System.out.println(first.getName() + " and " + last.getName() + " are buddies.");
}
```
正如前面讲到的，不能讲Pair\<Manager>传递给这个方法，这点很受限制。解决的方法很简单：使用通配符类型。

```
public static void printBuddies(Pair<? extends Employee> p)
```
类型Pair\<Manager>是Pair<? extends Employee>的子类型。

使用通配符会通过Pair\<? extends Employee>的引用破坏Pair<Manager>吗？

```
Pair<Manager> manager = new Pair<>(ceo, cfo);
Pair<? extends Employee> wildcard = manager; // OK
wildcard.setFirst(lowlyEmployee); // compile-time error
```
这可能不会引起破坏。对setFirst的调用有一个类型错误。Pair\<? extends Employee>类型中：

```
？extends Employee getFirst();
void setFirst(? extends Employee)
```
这样将不能调用setFirst方法。编译器只知道需要某个Employee的子类型，但不知道具体是什么类型的。他拒绝传递任何特定的类型。毕竟?不能用来匹配。

这就是引入有限定的通配符的关键之处。现在有办法区分安全的getter方法和不安全的setter方法了。

####通配符的超类型限定
通配符限定于类型变量限定十分类似，但它还有一个附加的能力，即可以指定一个**超类型限定(supertype bound)**，如下所示：

```
? super Manager
```
这个通配符限制为Manager的素有超类。(已有的super关键字十分准确的描述了这种联系)。这样可以为方法提供参数，但不能使用返回值。

```
void setFirst(? super Manager)
? super Manager getFirst()
```
可以使用任意Manager对象调用，而不能用Employee对象调用。然而，调用getFirst，返回的对象类型就不能得到保证了，只能把它付给一个Object。

**直观地讲，带有超类型限定的通配符可以向泛型对象写入，带有子类型限定的通配符可以从泛型对象读取。**

####无限定通配符
```
? getFirst()
void setFirst(?)
```
getFirst的返回值只能赋给一个Object。setFirst方法不能被调用，**甚至不能用Object调用**。Pair\<?>和Pair本质的不同在于：可以任意Object对象调用原始的Pair类的setObject方法。
> **注释：**可以调用setFirst(null)

这种类型对简单地操作非常有用。例如，下面的方法用来测试一个Pair是否包含一个null引用，它不需要实际的类型：

```
public static boolean hasNull(Pair<?> p) {
    return p.getFirst() == null || p.getLast() == null;
}
```
通过将hasNull转换成泛型方法，可以避免使用通配符类型：

```
public static <T> boolean hasNull(Pair<T> p)
```
但带有通配符的版本可读性更强。

####通配符捕获
```
public static void swap(Pair<?> p)
```
通配符不是类型变量，因此不能再编写代码时使用"?"作为一种类型。

```
? t = p.getFirst(); // ERROR
p.setFirst(p.getLast());
p.setLast(t);
```
可以使用写一个辅助方法swapHelper解决：

```
public static <T> void swapHelper(Pair<T> p) {
    T t = p.getFirst();
    p.setFirst(p.getLast());
    p.setLast(t);
}
```
注意swapHelper是一个泛型方法，而swap不是，它有固定的Pair<?>类型的参数。现在可以有swap调用swapHelper：

```
public static void swap(Pair<?> p) { swapHelper(p); }
```
这种情况下，swapHelper方法的参数T捕获通配符。他不知道是哪种类型的通配符，但是这是一个明确的类型，并且\<T> swapHelper的定义只有在T指出类型是才有明确的含义。

通配符播或只有在有许多限制的情况下才是合法的。编译器必须能够确信通配符表达的是单个、确定的类型。例如，ArrayLis\<Pair\<T>>中的T永远不能捕获ArrayList\<Pair\<?>>中的通配符。驻足列表可以保存两个Pair<?>，分别针对？的不同类型。
