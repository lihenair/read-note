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
```String middle = ArrayAlg.<String>getMiddle("Jonh", "Q", "Public");```。这种情况下，方法调用中可以省略<String>类型参数。编译器有足够的信息能够推断出所调用的方法。它用names的类型(即String[])与泛型类型T[]进行匹配并推断出T一定是String。也就是说可以调用
```
String middle = ArrayAlg.getMiddle(3.14, 1729, 0);```。

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
