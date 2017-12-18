# javascript的垃圾收集机制

　　javascript具有自动垃圾收集机制，执行环境会负责管理代码执行过程中使用的内存。在编写javascript程序时，开发人员不用再关心内存使用问题，所需内存的分配以及无用内存的回收完全实现了自动管理。下面将详细介绍javascript的垃圾收集机制

&nbsp;

### 原理

　　垃圾收集机制的原理很简单：找出那些不再继续使用的变量，然后释放其占用的内存，垃圾收集器会按照固定的时间间隔，或代码执行中预定的收集时间，周期性地执行这一操作

　　局部变量只在函数执行的过程中存在。而在这个过程中，会为局部变量在栈(或堆)内存上分配相应的空间，以便存储它们的值。然后在函数中使用这些变量，直到函数执行结束。此时，局部变量就没有存在的必要了。因此可以释放它们的内存以供将来使用。在这种情况下，很容易判断变量是否还有存在的必要；但并非所有情况下都这么容易就能得出结论

　　垃圾收集器必须跟踪哪个变量有用哪个变量无用，对于不再有用的变量打上标记，以备将来收回其所占用的内存。用于标识无用变量的策略通常有标记清除和引用计数两种

&nbsp;

### 标记清除

　　javascript中最常用的垃圾收集方式是标记清除(mark-and-sweep)，当变量进入环境(例如，在函数中声明一个变量)，就将这个变量标记为'进入环境'。从逻辑上讲，永远不能释放进入环境的变量所占用的内存，因为只要执行流进入相应的环境，就可能会用到它们。而当变量离开环境时，则将其标记为'离开环境'

　　可以用任何方式来标记变量。比如，可以通过翻转某个特殊的位来记录一个变量何时进入环境，或者使用一个'进入环境的'变量列表以及一个'离开环境'的变量列表来跟踪哪个变量发生了变化。说到底，如何标记变量其实并不重要，关键在于采取什么策略

　　垃圾收集器在运行的时候会给存储在内存中的所有变量都加上标记。然后，它会去掉环境中的变量以及被环境中的变量引用的变量的标记。而在此之后再被加上标记的变量将被视为准备删除的变量，原因是环境中的变量已经无法访问到这些变量了。最后，垃圾收集器完成内存清除工作，销毁那些带标记的值并回收它们所占用的内存空间

　　大多数浏览器实现使用的都是标记清除式的垃圾收集策略，只不过垃圾收集的时间互有不同

&nbsp;

### 引用计数

　　另一种不太常见的垃圾收集策略叫做引用计数(reference counting)

　　引用计数的含义是跟踪记录每个值被引用的次数。当声明了一个变量并将一个引用类型值赋给该变量时，则这个值的引用次数就是1，如果同一个值又被赋给另一个变量，则该值的引用次数加1。相反，如果包含对这个值的引用的变量又取得了另外一个值，则这个值的引用次数减1，当这个值的引用次数为0时，则说明没有办法再访问这个值了，因此就可以将其占用的内存空间回收回来。这样，当垃圾收集器下次再运行时，它就会释放那些引用次数为0的值所占用的内存

　　Netscape Navigator3.0是最早使用引用计数策略的浏览器，但很快它就遇到了一个严重的问题&mdash;&mdash;循环引用：对象A中包含一个指向对象B的指针，对象B中也包含一个指向对象A的指针

<div class="cnblogs_code">
<pre>function problem(){
    var objectA = new Object();
    var objectB = new Object();
    objectA.someOtherObject = objectB;
    objectB.anotherObject = objectA;
}</pre>
</div>

　　在这个例子中，objectA和objectB通过各自的属性相互引用，这两个对象的引用次数都是2。在采用标记清除策略的实现中，由于函数执行之后，这两个对象都离开了作用域，因此这种相互引用不是问题。但在采用引用计数策略的实现中，当函数执行完毕之后，objectA和objectB还将继续存在，因为它们的引用次数永远不会是0。假如这个函数被重复多次调用，就会导致大量内存得不到回收

　　IE中有一部分对象并不是原生javascript对象，例如，其BOM和DOM中的对象就是使用c++以COM(component Object Model 组件对象模型)对象的形式实现，而COM对象的垃圾回收机制采用的就是引用计数策略。因此，即使IE的javascript引擎是使用标记清除策略来实现的，但javascript访问的COM对象依然是基于引用计数策略的。换句话说，只要在IE中涉及COM对象，就会存在循环引用的问题

<div class="cnblogs_code">
<pre>var element = document.getElementById('some_element');
var myObject = new Object();
myObject.element = element;
element.someObject = myObject;</pre>
</div>

　　这个例子在一个DOM元素(element)与一个原生javascript对象(myObject)之间创建了循环引用。其中，变量myObject有一个名为element的属性指向element对象，而变量element也有一个属性名为someObject的属性指向myObject。由于存在这个循环引用，即使将例子中的DOM从页面中移除，它也永远不会被回收

　　为了避免类似这样的循环引用，最好是在不使用它们的时候手工断开原生javascript和DOM元素之间的连接

<div class="cnblogs_code">
<pre>myObject.element = null;
element.someObject = null; </pre>
</div>

　　将变量设置为null意味着切断变量与它此前引用的值之间的连接。当垃圾收集器下次运行时，就会删除这些值并回收它们占用的内存

　　为了解决此问题，IE9把BOM和DOM对象都转换成了真正的javascript对象

&nbsp;

### 性能问题

　　垃圾收集器是周期性运行的，而且如果为变量分配的内存数量很可观，那么回收工作量也是相当大的。在这种情况下，确定垃圾收集时间间隔是一个非常重要的问题

　　IE的垃圾收集器是根据内存分配量运行的。具体一点说，就是256个变量，4096个对象(或数组)字面量和数组元素(slot)或者64kb的字符串。达到上述任何一个临界值，垃圾收集器就会运行

　　这种实现方式的问题在于，如果一个脚本中包含那么多变量，那么该脚本很可能会在其生命周期中一直保有那么多的变量。而这样一来，垃圾收集器就不得不频繁地运行。结果，由此引发的严重性能问题促使IE7重写了其垃圾收集例程

　　IE7的javascript引擎的垃圾收集例程改变了工作方式：触发垃圾收集的变量分配、字面量和数组元素的临界值被调整为动态修正。IE7中的各项临界值在初始时与IE6相等。如果垃圾收集例程回收的内存分配量低于15%，则变量、字面量和数组元素的临界值就会加倍。如果例程回收了85%的内存分配量，则将各种临界值重置回默认值。这样，极大地提升了IE在运行包含大量javascript的页面时的性能

　　事实上，在有的浏览器中可以触发垃圾收集过程。在IE中，调用window.CollectGarbage()方法会立即执行垃圾收集

&nbsp;

### 内存管理

　　使用具备垃圾收集机制的javascript的主要问题在于：分配给web浏览器的可用内存数量通常要比分配给桌面应用程序的少，目的是防止运行javascript的网页耗尽全部系统内存而导致系统崩溃。内存限制问题不仅会影响给变量分配内存，同时还会影响调用栈以及在一个线程中能够同时执行的语句数量

　　因此，确保占用最少的内存可以让页面获得更好的性能。而优化内存占用的最佳方式是：为执行中的代码只保存必要的数据。一旦数据不再有用，最好通过将其值设置为null来释放其引用，这种做法叫解除引用(dereferencing)。这一做法适用于大多数全局变量和全局对象的属性，局部变量会在它们离开执行环境时自动被解除引用

<div class="cnblogs_code">
<pre>function createPerson(name){
    var localPerson = new Object();
    localPerson.name = name;
    return localPerson;
}

var globalPerson = createPerson('test');
globalPerson = null;</pre>
</div>

　　不过，要注意的是，解除一个值的引用并不意味着自动回收该值所占用的内存。解除引用的真正作用是让值脱离执行环境，以便垃圾收集器下次运行时将其回收
