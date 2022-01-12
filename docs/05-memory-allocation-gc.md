# 内存分配与回收策略

对象的内存分配，就是在堆上分配（也可能经过 JIT 编译后被拆散为标量类型并间接在栈上分配），对象主要分配在新生代的 Eden 区上，少数情况下可能直接分配在老年代，**分配规则不固定**，取决于当前使用的垃圾收集器组合以及相关的参数配置。

以下列举几条最普遍的内存分配规则，供大家学习。

## 对象优先在 Eden 分配

大多数情况下，对象在新生代 Eden 区中分配。当 Eden 区没有足够空间进行分配时，虚拟机将发起一次 Minor GC。

👇**Minor GC** vs **Major GC**/**Full GC**：

- Minor GC：回收新生代（包括 Eden 和 Survivor 区域），因为 Java 对象大多都具备朝生夕灭的特性，所以 Minor GC 非常频繁，一般回收速度也比较快。
- Major GC / Full GC: 回收老年代，出现了 Major GC，经常会伴随至少一次的 Minor GC，但这并非绝对。Major GC 的速度一般会比 Minor GC 慢 10 倍 以上。

> 在 JVM 规范中，Major GC 和 Full GC 都没有一个正式的定义，所以有人也简单地认为 Major GC 清理老年代，而 Full GC 清理整个内存堆。

## 大对象直接进入老年代

大对象是指需要大量连续内存空间的 Java 对象，如很长的字符串或数据。

一个大对象能够存入 Eden 区的概率比较小，发生分配担保的概率比较大，而分配担保需要涉及大量的复制，就会造成效率低下。

虚拟机提供了一个 -XX:PretenureSizeThreshold 参数，令大于这个设置值的对象直接在老年代分配，这样做的目的是避免在 Eden 区及两个 Survivor 区之间发生大量的内存复制。（还记得吗，新生代采用复制算法回收垃圾）

## 长期存活的对象将进入老年代

JVM 给每个对象定义了一个对象年龄计数器。当新生代发生一次 Minor GC 后，存活下来的对象年龄 +1，当年龄超过一定值时，就将超过该值的所有对象转移到老年代中去。

使用 `-XXMaxTenuringThreshold` 设置新生代的最大年龄，只要超过该参数的新生代对象都会被转移到老年代中去。

## 动态对象年龄判定

如果当前新生代的 Survivor 中，相同年龄所有对象大小的总和大于 Survivor 空间的一半，年龄 &gt;= 该年龄的对象就可以直接进入老年代，无须等到 `MaxTenuringThreshold` 中要求的年龄。

## 空间分配担保

JDK 6 Update 24 之前的规则是这样的：

在发生 Minor GC 之前，虚拟机会先检查**老年代最大可用的连续空间是否大于新生代所有对象总空间**， 如果这个条件成立，Minor GC 可以确保是安全的； 如果不成立，则虚拟机会查看 `HandlePromotionFailure` 值是否设置为允许担保失败， 如果是，那么会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小， 如果大于，将尝试进行一次 Minor GC,尽管这次 Minor GC 是有风险的； 如果小于，或者 `HandlePromotionFailure` 设置不允许冒险，那此时也要改为进行一次 Full GC。

JDK 6 Update 24 之后的规则变为：

只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小，就会进行 Minor GC，否则将进行 Full GC。

通过清除老年代中的废弃数据来扩大老年代空闲空间，以便给新生代作担保。

这个过程就是分配担保。

---

👇 总结一下有哪些情况可能会触发 JVM 进行 Full GC。

1. **`System.gc()` 方法的调用**
   此方法的调用是建议 JVM 进行 Full GC，注意这**只是建议而非一定**，但在很多情况下它会触发 Full GC，从而增加 Full GC 的频率。通常情况下我们只需要让虚拟机自己去管理内存即可，我们可以通过 -XX:+ DisableExplicitGC 来禁止调用 `System.gc()`。
1. **老年代空间不足**
   老年代空间不足会触发 Full GC 操作，若进行该操作后空间依然不足，则会抛出如下错误：`java.lang.OutOfMemoryError: Java heap space`
1. **永久代空间不足**
   JVM 规范中运行时数据区域中的方法区，在 HotSpot 虚拟机中也称为永久代（Permanet Generation），存放一些类信息、常量、静态变量等数据，当系统要加载的类、反射的类和调用的方法较多时，永久代可能会被占满，会触发 Full GC。如果经过 Full GC 仍然回收不了，那么 JVM 会抛出如下错误信息：`java.lang.OutOfMemoryError: PermGen space `
1. **CMS GC 时出现 `promotion failed` 和 `concurrent mode failure`**
   promotion failed，就是上文所说的担保失败，而 concurrent mode failure 是在执行 CMS GC 的过程中同时有对象要放入老年代，而此时老年代空间不足造成的。
1. **统计得到的 Minor GC 晋升到旧生代的平均大小大于老年代的剩余空间。**

[Major GC和Full GC的区别是什么？触发条件呢？](https://www.zhihu.com/question/41922036/answer/93079526)

## Blog

[java的gc为什么要分代？](https://www.zhihu.com/question/53613423/answer/135743258)

+ 所有[Java线程](https://www.zhihu.com/search?q=Java线程&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A135743258})当前活跃的栈帧里指向GC堆里的对象的引用；换句话说，当前所有正在被调用的方法的引用类型的参数/[局部变量](https://www.zhihu.com/search?q=局部变量&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A135743258})/临时值。

+ VM的一些静态数据结构里指向GC堆里的对象的引用，例如说HotSpot VM里的Universe里有很多这样的引用。

+ JNI handles，包括[global handles](https://www.zhihu.com/search?q=global+handles&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A135743258})和local handles

+ （看情况）所有当前被加载的Java类

+ （看情况）Java类的引用类型[静态变量](https://www.zhihu.com/search?q=静态变量&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A135743258})

+ （看情况）Java类的运行时常量池里的引用类型常量（String或Class类型）

+ （看情况）String常量池（StringTable）里的引用

分代式GC对GC roots的定义有什么影响呢？

答案是：分代式GC是一种部分收集（partial collection）的做法。在执行部分收集时，从GC堆的非收集部分指向收集部分的引用，也必须作为GC roots的一部分。

具体到分两代的分代式GC来说，如果第0代叫做young gen，第1代叫做[old gen](https://www.zhihu.com/search?q=old+gen&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A135743258})，那么如果有minor GC / young GC只收集young gen里的垃圾，则young gen属于“收集部分”，而old gen属于“非收集部分”，那么从old gen指向young gen的引用就必须作为minor GC / young GC的GC roots的一部分。

继续具体到HotSpot VM里的分两代式GC来说，除了old gen到young gen的引用之外，有些带有弱引用语义的结构，例如说记录所有当前被加载的类的SystemDictionary、记录字符串常量引用的StringTable等，在young GC时必须要作为strong GC roots，而在收集整堆的full GC时则不会被看作strong GC roots。

那么分代有什么好处？

对传统的、基本的GC实现来说，由于它们在GC的整个工作过程中都要“stop-the-world”，如果能想办法缩短GC一次工作的时间长度就是件重要的事情。如果说收集整个GC堆耗时太长，那不如只收集其中的一部分？

于是就有好几种不同的划分（partition）GC堆的方式来实现部分收集，而分代式GC就是这其中的一个思路。

这个思路所基于的基本假设大家都很熟悉了：weak generational hypothesis——大部分对象的[生命期](https://www.zhihu.com/search?q=生命期&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A135743258})很短（die young），而没有die young的对象则很可能会存活很长时间（live long）。

这是对过往的很多应用行为分析之后得出的一个假设。基于这个假设，如果让新创建的对象都在young gen里创建，然后频繁收集young gen，则大部分垃圾都能在young GC中被收集掉。由于young gen的大小配置通常只占整个GC堆的较小部分，而且较高的对象死亡率（或者说较低的对象存活率）让它非常适合使用[copying算法](https://www.zhihu.com/search?q=copying算法&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A135743258})来收集，这样就不但能降低单次GC的时间长度，还可以提高GC的工作效率。

[Major GC和Full GC的区别是什么？触发条件呢？](https://www.zhihu.com/question/41922036/answer/93079526)

针对HotSpot VM的实现，它里面的GC其实准确分类只有两大种：

- Partial GC：并不收集整个GC堆的模式

- - Young GC：只收集young gen的GC
  - Old GC：只收集[old gen](https://www.zhihu.com/search?q=old+gen&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A93079526})的GC。只有CMS的concurrent collection是这个模式
  - Mixed GC：收集整个young gen以及部分old gen的GC。只有G1有这个模式

- Full GC：收集整个堆，包括young gen、old gen、[perm gen](https://www.zhihu.com/search?q=perm+gen&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A93079526})（如果存在的话）等所有部分的模式。

[YGC问题排查，又让我涨姿势了](https://mp.weixin.qq.com/s/-8xYoAkBUoavcSl69I0XJw)

各个问题不错，值得反思

可作为YGC时GC Root的对象包括以下几种：

> 1、虚拟机栈中引用的对象
>
> 2、方法区中静态属性、常量引用的对象
>
> 3、本地方法栈中引用的对象
>
> 4、被Synchronized锁持有的对象
>
> 5、记录当前被加载类的SystemDictionary
>
> 6、记录字符串常量引用的StringTable
>
> 7、存在跨代引用的对象
>
> 8、和GC Root处于同一CardTable的对象

针对跨代引用的情况，老年代的对象A也必须作为GC Root的一部分，但是如果每次YGC时都去扫描老年代，肯定存在效率问题。在HotSpot JVM，引入卡表（Card Table）来对跨代引用的标记进行加速

[JVM垃圾回收18问](https://mp.weixin.qq.com/s/XsZUF2nBUSEJoGIA8RimJw)

+ young gc 触发条件是什么？

大致上可以认为在年轻代的 eden 快要被占满的时候会触发 young gc。

为什么要说大致上呢？因为有一些收集器的回收实现是在 full gc 前会让先执行以下 young gc。

比如 Parallel Scavenge，不过有参数可以调整让其不进行 young gc。

可能还有别的实现也有这种操作，不过正常情况下就当做 eden 区快满了即可。

eden 快满的触发因素有两个，一个是为对象分配内存不够，一个是为 TLAB 分配内存不够。

+ TLAB

TLAB（Thread Local Allocation Buffer），为一个线程分配的内存申请区域。

这个区域只允许这一个线程申请分配对象，允许所有线程访问这块内存区域

如果这块区域用完了再去申请即可。不过每次申请的大小不固定，会根据该线程启动到现在的历史信息来调整，比如这个线程一直在分配内存那么 TLAB 就大一些，如果这个线程基本上不会申请分配内存那 TLAB 就小一些

还有 TLAB 只能分配小对象，大的对象还是需要在共享的 eden 区分配。

所以总的来说 TLAB 是为了避免对象分配时的竞争而设计的

+ PLAB

PLAB 即 Promotion Local Allocation Buffers。用在年轻代对象晋升到老年代时。

在多线程并行执行 YGC 时，可能有很多对象需要晋升到老年代，此时老年代的指针就“热”起来了，于是搞了个 PLAB。

先从老年代 freelist（空闲链表） 申请一块空间，然后在这一块空间中就可以通过指针加法（bump the pointer）来分配内存，这样对 freelist 竞争也少了，分配空间也快了

+ 新生代的 GC 如何避免全堆扫描？

在常见的分代 GC 中就是利用记忆集来实现的，记录可能存在的老年代中有新生代的引用的对象地址，来避免全堆扫描