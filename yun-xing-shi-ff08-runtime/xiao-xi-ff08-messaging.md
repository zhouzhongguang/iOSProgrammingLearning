# 消息传递（Messaging）

这个章节描述消息表达式如何被转换成objc\_msgSend函数调用，以及如何通过名称引用方法。然后解释如何利用objc\_msgSend，以及如果需要你如何绕过动态绑定。

## objc\_msgSend函数（The objc\_msgSend Function）

在Objective-C，消息直到运行时才被绑定到方法实现。编译器转换一个消息表达式，

```
[receiver message]
```

为一个消息函数_objc\_msgSend_的调用。这个函数将接受者和消息中提到的方法实现的名称（即方法选择器）作为其两个主要参数：

```
objc_msgSend(receiver, selector)
```

消息中传递的任何参数也将传递给_objc\_msgSend_：

```
objc_msgSend(receiver, selector, arg1, arg2, ...)
```

消息传递功能完成动态绑定所需的一切：

* 它首先找到选择器引用的过程（方法实现-method implementation）。由于同样的方法可以通过不同的类产生不同的实现，它发现的精确过程依赖于接受者的类。
* 然后它调用过程，将接收的对象（指向其数据的指针）以及为该方法指定的任何参数传递给过程。
* 最后，它传递过程的返回值作为它自己的返回值。

> 注意：编译器生成对消息传递函数的调用。你不应该在你写的代码中直接调用它。

消息传递的关键在于编译器为每个类和对象构建的结构。每个类结构包含着两个基本要素：

* 一个指向超类（superclass）的指针。
* 一个类调度表（_dispatch table_）。该表具有将方法选择器与它们识别的方法的类特定地址相关联的条目。_setOrigin::_方法的选择器与_setOrigin::_（实现过程）的地址相关联，_display_方法的选择器跟_display_的地址相关联，以此类推。

当一个新对象创建，它的内存被分配，并且它的实例变量被初始化。对象的变量中的第一个是指向其类结构的指针。这个名为isa的指针为对象提供对其类的访问权限，并通过该类访问所有它继承的类。

> 注意：尽管不是该语言的一部分，对象需要使用isa指针才能与Objective-C运行时系统一起工作。一个对象需要与结构_objc\_object_（在objc/objc.h中定义）在结构定义的任何字段中“等价”。但是，你很少需要创建自己的根对象，并且从NSObject或NSProxy继承的对象自动具有isa变量。

这些类和对象结构的元素如下图所示。

![](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Art/messaging1.gif)

当消息发送到一个对象时，消息传递函数跟随对象的isa指针，指向查找调度表中的方法选择器的类结构。如果在那里没能发现选择器，_objc\_msgSend_跟随指针到超类，并尝试在该类的调度表中发现选择器。连续失败导致_objc\_msgSend_爬上类层次结构，直到它到达NSObject类。一旦他定位到选择器，该函数将调用在表中输入的方法并将接收对象的数据结构传递给它。

这就是运行时选择方法实现的方式--或者，在面向对象编程术语中，方法是动态绑定到消息的。

为了加速消息传递过程，运行时系统缓存选择器和方法的地址。每个类都有独立的缓存，并且它可以包含继承的方法的选择器以及类中定义的方法。查找调度表之前，消息传递首先例行检查接收对象的类的缓存（理论上曾经使用过的方法可能会再次使用）。如果方法选择器在缓存中，消息传递只比函数调用慢一点。一旦一个程序已经运行足够长的时间来“加热”其缓存，几乎它发送的所有消息都能找到缓存的方法。程序运行时，缓存动态增长以适应新的消息。

## 使用隐藏的参数（Using Hidden Arguments）

当_objc\_msgSend_发现实现一个方法的过程，它调用过程并把消息中的所有参数传递给它。它也传递给过程两个隐藏的参数：

* 接收对象
* 方法的选择器

这些参数为每个方法提供了关于调用它的消息表达式的两半的确切信息。它们被称作“隐藏的”，因为它们没有在方法定义的源码中声明。它们在代码编译时被插入到实现中。

尽管这些方法没有被明确的声明，源码依然可以引用他们（就像它可以引用接收对象的实例变量一样）。方法引用接收对象称为self，它自己的选择器作为\_cmd。下面的例子中，\_cmd指向strange方法的选择器，self指向接收strange方法的对象自身。

```
- strange
{
    id  target = getTheReceiver();
    SEL method = getTheMethod();

    if ( target == self || method == _cmd )
        return nil;
    return [target performSelector:method];
}
```

两个参数中self更有用。实际上，接收对象的实例变量可用以方法定义。

## 获取方法的地址（Getting a Method Address）

规避动态绑定的唯一方法是获取方法的地址并直接调用就好像它是个函数。在特定的方法将连续执行多次并且你希望避免每次执行该方法时消息传递的开销的情况下，这可能是合适的。

使用NSObject类中定义的方法_methodForSelector:_，你可以获得一个指向实现方法过程的指针，然后使用该指针调用过程。_methodForSelector:_方法返回的指针必须小心转换为正确的函数类型。返回类型和参数类型也必须包括在转换中。

下面的例子展示了如何调用实现setFilled:方法的过程：

```
void (*setter)(id, SEL, BOOL);
int i;

setter = (void (*)(id, SEL, BOOL))[target
    methodForSelector:@selector(setFilled:)];
for ( i = 0 ; i < 1000 ; i++ )
    setter(targetList[i], @selector(setFilled:), YES);
```

传递给过程的前面两个参数是接收对象（self）和方法选择器（\_cmd）。这些参数在方法语法中是隐藏的，但是当方法作为函数被调用时必须明确。

使用_methodForSelector:_规避动态绑定节省了消息传递所需的大部分时间。然而，只有当在特定消息重复多次的情况下，节省才会显著，如上面所示的for循环。

请注意，_methodForSelector:_是由Cocoa运行时系统提供的；它不是Objective-C语言自己的特性。

