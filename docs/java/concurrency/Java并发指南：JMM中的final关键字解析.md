**本文转载自互联网，侵删**

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章首发于我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《Java并发指南》其中一篇，本文大部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何全面深入地学习Java并发技术，从Java多线程基础，再到并发编程的基础知识，从Java并发包的入门和实战，再到JUC的源码剖析，一步步地学习Java并发编程，并上手进行实战，以便让你更完整地了解整个Java并发编程知识体系，形成自己的知识框架。

为了更好地总结和检验你的学习成果，本系列文章也会提供一些对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->

## 一、properly constructed / this对象逸出
在开始讲之前final之前，先了解一个概念，叫做 “properly constructed”。其含义是：在构造器创建对象的过程中，正在被创建的对象的引用没有发生 “逸出(escape)” 。
````
public Test {
private final int x;
private int y;
private static Test instance;

	public Test(int x, int y) {
		this.x = x;
		this.y = y;
		instance = this;
	}
}
````
在上面的例子中，在构造器中把正在创建的对象赋值给了一个静态变量instance，这种行为就叫“逸出”。

## 二、对象的安全发布
所谓对象的安全发布，意思就是构建一个被完整初始化的对象，防止一个未完全初始化的对象被访问。

看下面的例子：
````
public class Test {
static MyObj obj;

	static class MyObj {
		int a, b, c, d;
		MyObj() {
			a = 1;
			b = 1;
			c = 1;
			d = 1;
		}
	}
	
	// Thread 1
	public static void init() {
		obj = new MyObj();
	}
	
	// Thread 2
	public static void f() {
		if (obj == null) return;
		if (obj != null) {
			System.out.println(obj.a + obj.b + obj.c + obj.d);
		}
	}

}
````

上述代码中，Thread 2 打印出的数将会是几呢？事实上，可能是0, 1, 2, 3, 4 中的任意一个值。

如果打印出的数不是4，那么就说明线程2读到了未完全初始化的MyObj对象。

为什么会出现这种情况呢？是因为 obj = new MyObj() 这个操作底层实际上分为以下3个步骤：

在堆上给MyObj对象分配空间，对象的字段置为默认值
执行构造器中的初始化语句进行初始化
将堆上的MyObj对象的引用赋值给obj
其中，步骤2和3是可能发生重排序的 (StoreStore重排序)，因此就Thread 2就可能看到 obj != null，但是a,b,c,d还没全部赋值完成的情况，也就是 Thread 1 不安全的发布了 MyObj 对象。

扩展一下，在实现单例模式时，会什么要double checked且加上volatile，正是这个原因，防止不安全的发布。

笔者在 x86 平台的 HotSpot 虚拟机中使用 jcstress 工具对不安全发布现象进行了测试，发现类似于上述的代码在 x86 平台的 HotSpot 虚拟机中并不会出现为完全初始化的情况，出现这种情况就说明，x86 平台的 HotSpot 虚拟机实现中，编译器没有对上述步骤2和3进行重排序，而x86的内存模型又保证了x86不会发生StoreStore重排序，因此上述步骤2和3不会发生重排序，进而始终可以读到完全初始化过的对象。但是，在其他硬件平台或其他JVM中就未必如此了。 我们的Java代码不应该基于特定硬件平台或虚拟机实现之上，而是应该遵循JLS和JMM的规范！

## 三、 final 关键字的内存语义

1. 可见性
   摘自[1]：

The values for an object’s final fields are set in its constructor. Assuming the object is constructed “correctly”, once an object is constructed, the values assigned to the final fields in the constructor will be visible to all other threads without synchronization. In addition, the visible values for any other object or array referenced by those final fields will be at least as up-to-date as the final fields.

在JMM中规定，如果我们在构造器中对final字段进行初始化，并且构造器中没有发生this对象的逸出，那么无需任何同步措施，即可确保其他线程可以看到构造器中初始化给final字段的值。此外，如果这个final字段是一个引用类型，那么可以确保该引用类型对象引用到的对象或数组的内容都是至少和final字段一样新的值。

这里的“至少和final字段一样新的值”，意思也就是 up to date as of the end of the object’s constructor ([1])，也就是说，对于引用类型的final变量，其他线程至少能够读到构造器结束时，这个final类型引用变量的成员的状态。

实际上，final 关键字对可见性的影响在Java语言规范的 17.5.1. Semantics of final Fields 一节也作出了正式规范。针对final关键字，引入了两个偏序关系——Dereference Chain和Memory Chain，借助这两个偏序关系，可以在不同线程之间的 w ww 和 r 2 r2r2 操作之间建立 happens before 关系，如下图所示：


而 happens before 关系又隐含着可见性，所以 w ww 写的内容对于 r 2 r2r2 是可见的。结合上图 w ww 到 r 2 r2r2 的关系链，我们也不难理解为什么之前说 “对于引用类型的final变量，其他线程至少能够读到构造器结束时，这个final类型引用变量的成员的状态” 了，借助上述关系同样可以建立修改final字段成员 和 读取final字段成员之间的happens before 关系。

2. 有序性

注意：重排序是针对单个线程（单个CPU）而言的。

以下摘自文档 [2]：

Loads and Stores of final fields act as “normal” accesses with respect to locks and volatiles, but impose two additional reordering rules:

对final字段的load和store操作和对普通字段的load和store操作几乎一样，只不过针对final字段存在下面两条额外的重排序规则：

规则1

【① A store of a final field】 (inside a constructor) and, 【② if the field is a reference, any store that this final can reference】, cannot be reordered with 【③ a subsequent store (outside that constructor) of the reference to the object holding that field into a variable accessible to other threads】. For example, you cannot reorder
x.finalField = v; ... ; sharedRef = x;

This comes into play for example when inlining constructors, where “...” spans the logical end of the constructor. You cannot move stores of finals within constructors down below a store outside of the constructor that might make the object visible to other threads. (As seen below, this may also require issuing a barrier). Similarly, you cannot reorder either of the first two with the third assignment in:
v.afield = 1; x.finalField = v; ... ; sharedRef = x;

直接看上面的话有点难懂，下面举一个例子：
````
class Apple {
private String color;

	public Apple(String color) { this.color = color; }
}

public class Test {
static Test instance;
final int a;
final Apple apple;

	public Test() { 
		a = 10;
		apple = new Apple("red"); 
	}
	
	public static void init() {
		instance = new Test();
	}
}
````

其实， x.finalField = v; ... ; sharedRef = x; 中的 x 在构造器中可以理解为 this，在 init() 方法中可以理解为new Test() 返回的对象。

上述英文文档中：

操作①是在构造器中对final字段 (基本类型和引用类型) 的赋值操作，对应a = 10，它不能和操作③进行重排序
操作②也是指构造器中的操作，在①的基础上，如果final字段是一个引用类型，那么所有这个final字段引用到的store操作，都不能和操作③重排序。例如上面代码中的 apple 就是一个 final 类型应用变量，那么通过apple可以引用到color，则对apple.color的赋值操作不能和③进行重排序。
操作③是在构造器外，将包含final字段的对象赋值给一个其他线程可以访问到的变量。在上述代码中，也就对应init()方法中的instance = new Test()，因为new Test()对象包含final字段a，且字段instance可以被其他线程访问到。
实现这一条重排序规则可以通过在构造器结束位置插入StoreStore屏障来实现。

规则2
The initial load (i.e., the very first encounter by a thread) of a final field cannot be reordered with the initial load of the reference to the object containing the final field. This comes into play in:
x = sharedRef; ... ; i = x.finalField;
A compiler would never reorder these since they are dependent, but there can be consequences of this rule on some processors.

再看个例子：
````
public class Test {
static Test instance;
final int a;

	public Test() { a = 10; }
	
	public static void init() {
		instance = new Test();
	}
	
	public static void read() {
		if (obj != null) {
			System.out.println(obj.a);
		}
	}
}
````
这条规则的意思是，通过对象访问其final字段(obj.a)这一操作 和 该操作之前第一次访问该对象的操作 (if (obj != null)中读取obj的操作) 是不能重排序的。

Java编译器不会对不会对上述两个操作进行重排序，因为对编译器来说这两个操作(load(x)和load(x.field))之间存在依赖。但是对于处理器来说，它俩都是load操作，所以在允许LoadLoad重排序的处理器上，这两个操作是可能被重排序的，此时就需要加上LoadLoad屏障。

总结
final字段的重排序规则总结如下：

构造器内final字段的写 和 构造器外将包含该final字段的对象赋值给一个其他线程能访问到的变量 这两个操作不能重排序
加入final字段是一个引用类型，那么构造器内对该final引用类型字段的成员的写 和 构造器外将包含该final字段的对象赋值给一个其他线程能访问到的变量 这两个操作之间不能重排序
构造器外，对于包含final字段的对象的读 和 对final字段的成员的读 不能重排序

## 四、HotSpot VM中对final内存语义的实现
[3] 在HotSpot源码的 parse1.cpp 中：
````
//------------------------------do_exits---------------------------------------
void Parse::do_exits() {
// ...
if (method()->is_initializer() &&
(wrote_final() ||
PPC64_ONLY(wrote_volatile() ||)
(AlwaysSafeConstructors && wrote_fields()))) {
_exits.insert_mem_bar(Op_MemBarRelease, alloc_with_final());
// ...
do_exists() 函数在构造器退出时会执行，显然它会使用 wrote_final() 判断构造器内是不是存在final类型的写，如果是的话，则插入一个内存屏障 Op_MemBarRelease，这个屏障实际上对应 LoadStore 和 StoreStore。插入 LoadStore 屏障是为了照顾这样一种特殊情况：final 字段的值依赖于另外的字段。例如x.finalField = x.normalField + 1; ...; sharedRef = x;，这里插入LoadStore 就是为了防止对 x.normalField 的读操作和sharedRef = x 发生重排序。
````

 总结
final关键字的内存语义相较于volatile等关键字还是很难理解的，以上的内容如有表述不恰当之处还请指正。

总结
final关键字的内存语义相较于volatile等关键字还是很难理解的，以上的内容如有表述不恰当之处还请指正。

# 参考文献
Java Concurrency in Practice

JSR 133 (Java Memory Model) FAQ

Java Concurrency in Practice

The JSR-133 Cookbook for Compiler Writers
Intel® 64 and IA-32 ArchitecturesvSoftware Developer’s Manual Volume 3A: System Programming Guide, Part 1