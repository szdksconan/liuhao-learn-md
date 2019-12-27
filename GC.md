# 垃圾回收我们关注的三个点 
1. 垃圾回收发生在哪里
2. 哪些对象可以被回收
3. 怎么回收

## 1.回收发生在哪里 
- 在jvm区域中，程序计数器、本地方法栈、虚拟机方法栈是线程私有的，他们会随着线程的创建而创建、销毁而销毁；栈中的栈帧随著方法的进入和退出 进行入栈和出栈操作；每个栈帧分配的内存在类结构确认了就定下来了已知了，因此这些区域分配和回收都具有确定性。
- 需要关注的就是堆和方法区的内存了，堆中回收的主要是对象、方法区是废弃的常量和无用的类



## 2.哪些对象可以回收
般一个对象不再被引用，就代表改对象可以被回收,2种算法来判断:
1. 引用计数算法：对象引用计数器在对象被引用就+1引用失效就-1，当对象引用计数器的值为0则表示可以被回收，但这种算法在对象互相循环引用时存在问题
2. 可达性分析算法：GC roots 是所有对象的哦根对象，jvm加载时会创建一些普通正常的供引用的对象，这些对象作为正常对象的起始点，在垃圾回收时候 会从GC ROOTS 向下所有，当一个对象到GC ROOTS 没有任何一个引用链相连时，我们就认为它不可用。


## 3.怎么回收
### 垃圾回收具有两大特性
1. 自动性：jvm提供一个系统级的线程来跟踪分配出去的内存，当jvm处于空闲循环时，垃圾收集器线程会检测分配出去的内存，当内存处于空闲时就回收
2. 不可确定性：没有被引用的对象是否会被立即回收？这个是不可预期的。垃圾回收线程是自动执行，我们无法强制执行它，system.gc方法只是“建议”jvm执行垃圾回收，至于什么时间执行，执不执行不可预期




## GC 算法
### 常用几种
1. 标记-清除
2. 复制
3. 标记整理
4. 分代收集算法


## GC收集器

1. serial new/serial old回收器 复制算法/标记-整理算法 单线程回收、简单高效、但是停顿明显 。
2. parnew new /parold old回收器 复制算法/标记-整理算法 多线程回收、降低了停顿时间、但是有了上下文切换的消耗
3. parallei scavenge回收器 复制算法 并行回收、追求吞吐量、高效利用CPU 之前提到过，在进行GC的过程中要“Stop the world”，停顿时间越短当然越好，很多垃圾回收器（包括前两个）关注的就是如何提高停顿时间。而Parallel GC关注的则是吞吐量。它关注的是垃圾回收的整体耗时，如果垃圾回收所占用的整体耗时较短，则吞吐量高。
4. CMS 回收器 （Concurrent Mark Sweep） 标记-清理算法 老年代回收器，高并发低停顿，追求最短时间的GC停顿，CPU占用率高，响应快、挺短时间短。![image](https://static001.geekbang.org/resource/image/50/aa/500c2f0e112ced378fd49a09c61c5caa.jpg)
    - 初始化标记 stw 通过可达性算法 标记 GC ROOTS 可以直接关联的对象（这里注意是直接关联，为并发标记打下基础）
    - 并发标记 和用户线程并行 进行GC ROOT TRACING 由之前直接关联的对象出发 找到、标记处GC ROOTS间接关联的所有存活对象（并行期间发生变化的对象card标识变为dirty、用于预处理重新扫描标记。
    - 并发预处理 重新扫描标识为dirty的对象、分析其可达性、标记存活的对象(老年代的机制与一个叫CARD TABLE的东西,这个东西其实就是个数组,数组中每个位置存的是一个byte,CMS将老年代的空间分成大小为512bytes的块，card table中的每个元素对应着一个块,并发标记时，如果某个对象的引用发生了变化，就标记该对象所在的块为  dirty card并发预清理阶段就会重新扫描该块，将该对象引用的对象标识为可达)
    - 可终止的并发预处理 这个阶段算是一种优化策略，在重复做之前的事的时候在等待中止，因为重新标记要stw，为了尽可能的减少stw的时间，就要直接缩小需要分析的范围，所以这里在等待一次young 的 minor GC 这样年轻代所有的对象将会进去sur区减少检索范围，这里可以设置2个参数，CMSScheduleRemarkEdenSizeThreshold、CMSScheduleRemarkEdenPenetration，默认值分别是2M、50%。两个参数组合起来的意思是预清理后，eden空间使用超过2M时启动可中断的并发预清理（CMS-concurrent-abortable-preclean），直到eden空间使用率达到50%时中断，进入remark阶段；还有CMS提供了一个参数CMSMaxAbortablePrecleanTime ，默认为5S，只要到了5S，不管发没发生Minor GC，有没有到CMSScheduleRemardEdenPenetration都会中止此阶段，进入remark，这个时候CMS提供CMSScavengeBeforeRemark参数，使remark前强制进行一次Minor GC，这样做利弊都有，好的一面是减少了remark阶段的停顿时间，坏的一面是Minor GC后紧跟着一个remark pause，如此一来，停顿时间也比较久。
    - 重新标记 停止用户线程，重新扫描堆中的对象进行可达性分析，有了前面 的基础，这个阶段就会很快了，并且是多线程处理。    
    - 并发清理 启动用户线程，清理掉没有被标记的对象
    - 并发重置  CMS清除内部状态，为下次回收做准备。
5. G1回收器 标记-整理+复制算法 高并发、低停顿可预测停顿时间。G1是一个分代收集器，既负责年轻代，也负责老年代，以往的收集器用的是虚拟的虚拟内存地址，G1则不一样，它把堆内存划分成了很多个region，每个region占用连续的虚拟内存地址，每代占用N个不连续的region。![image](https://static001.geekbang.org/resource/image/f8/be/f832278afd5cdb94decd1f6826056dbe.jpg)G1 分为 Young GC、Mix GC 以及 Full GC。G1 Young GC 主要是在 Eden 区进行，当 Eden 区空间不足时，则会触发一次 Young GC。将 Eden 区数据移到 Survivor 空间时，如果 Survivor 空间不足，则会直接晋升到老年代。此时 Survivor 的数据也会晋升到老年代。Young GC 的执行是并行的，期间会发生 STW。当堆空间的占用率达到一定阈值后会触发 G1 Mix GC（阈值由命令参数 -XX:InitiatingHeapOccupancyPercent 设定，默认值 45），Mix GC 主要包括了四个阶段，其中只有并发标记阶段不会发生 STW，其它阶段均会发生 STW。![image](https://static001.geekbang.org/resource/image/b8/2f/b8090ff2c7ddf54fb5f6e3c19a36d32f.jpg)G1 和 CMS 主要的区别在于：CMS 主要集中在老年代的回收，而 G1 集中在分代回收，包括了年轻代的 Young GC 以及老年代的 Mix GC；G1 使用了 Region 方式对堆内存进行了划分，且基于标记整理算法实现，整体减少了垃圾碎片的产生；在初始化标记阶段，搜索可达对象使用到的 Card Table，其实现方式不一样。这里我简单解释下 Card Table，在垃圾回收的时候都是从 Root 开始搜索，这会先经过年轻代再到老年代，也有可能老年代引用到年轻代对象，如果发生 Young GC，除了从年轻代扫描根对象之外，还需要再从老年代扫描根对象，确认引用年轻代对象的情况。这种属于跨代处理，非常消耗性能。为了避免在回收年轻代时跨代扫描整个老年代，CMS 和 G1 都用到了 Card Table 来记录这些引用关系。只是 G1 在 Card Table 的基础上引入了 RSet，每个 Region 初始化时，都会初始化一个 RSet，RSet 记录了其它 Region 中的对象引用本 Region 对象的关系。除此之外，CMS 和 G1 在解决并发标记时漏标的方式也不一样，CMS 使用的是 Incremental Update 算法，而 G1 使用的是 SATB 算法。首先，我们要了解在并发标记中，G1 和 CMS 都是基于三色标记算法来实现的：黑色：根对象，或者对象和对象中的子对象都被扫描；灰色：对象本身被扫描，但还没扫描对象中的子对象；白色：不可达对象。基于这种标记有一个漏标的问题，也就是说，当一个白色标记对象，在垃圾回收被清理掉时，正好有一个对象引用了该白色标记对象，此时由于被回收掉了，就会出现对象丢失的问题。为了避免上述问题，CMS 采用了 Incremental Update 算法，只要在写屏障（write barrier）里发现一个白对象的引用被赋值到一个黑对象的字段里，那就把这个白对象变成灰色的。而在 G1 中，采用的是 SATB 算法，该算法认为开始时所有能遍历到的对象都是需要标记的，即认为都是活的。G1 具备 Pause Prediction Model ，即停顿预测模型。用户可以设定整个 GC 过程中期望的停顿时间，用参数 -XX:MaxGCPauseMillis 可以指定一个 G1 收集过程的目标停顿时间，默认值 200ms。G1 会根据这个模型统计出来的历史数据，来预测一次垃圾回收所需要的 Region 数量，通过控制 Region 数来控制目标停顿时间的实现。
    


##  GC衡量指标
1. 吞吐量
2. 停顿时间
3. 回收效率



## GC MINOR GC 和 FULL GC
1. MINOR GC：典型的标记-清除-整理算法 只是这里整理是eden、fromSpace 到 toSapce，当eden没有足够的空间时触发minor GC 把eden 和 fromSpace存活对象过度到toSpace里面，当对象年龄超过既定年龄(默认15岁，配置参数-XX:MaxTenuringThreshold)则会被晋升到old，这里强调一点 当对象足够大则因为委派机制直接进入 old 
2. FULL GC(jstat -gcutil pid查询GC频率): 
    - 触发条件(后面2种也是old空间不足)：
        - 调用System.gc时，系统建议执行Full GC，但是不必然执行
        - 老年代空间不足
        - 方法区空间不足
        - 通过Minor GC后进入老年代的平均大小大于老年代的可用内存
        - 由Eden区、From Space区向To Space区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小
        - 执行了jmap -histo:live pid命令 //这个会立即触发fullgc
    - 对象进入old的条件
        - 假如进行Minor GC时发现，存活的对象在ToSpace区中存不下，那么把存活的对象存入老年代
        - 大对象直接进入老年代(这个值可以通过PretenureSizeThreshold这个参数进行设置，默认3M)
        - 长期存活的对象将进入老年代(默认15岁，配置参数-XX:MaxTenuringThreshold)
        - 动态对象年龄判定:如果在From空间中，相同年龄所有对象的大小总和大于From和To空间总和的一半，那么年龄大于等于该年龄的对象就会被移动到老年代，而不用等到15岁(默认)
    - 因为以上原因进入old区域的存活对象，但是old区域没有足够的空间来容纳这些对象就会触发fullGC,fullGC会对整个heap进行GC，GC后如果还是无法分配空间给对象 则会抛出 outOfMemoryError 
    - 空间分配担保：简言之就是 old区的连续空间>young区的所有对象的大小则可以进行minorGC，如果不满足就要查看 HandlePromotionFailure是否为ture，为true则允许担保失败，继续进行空间大小和对象的比较，弱大于则进行minor GC 小于则进行FULLGC；如果HandlePromotionFailure是否为false,则不允许冒险直接进行FULLGC；说到底就是开起后再给一次检索的机会。


## GC调优策略（ 抛开单纯的young的eden sur 和 old 的大小调节)
1. CMS的几个问题：
    - 浮动垃圾：在并行清理是产生的新的对象，所以说CMS内存回收就要考虑这部分多余的内存，预留内存空间，CMS提供了CMSInitiatingOccupancyFraction参数来设置老年代空间使用百分比,达到百分比就进行垃圾回收，这个参数默认是92%，参数选择需要看具体的应用场景，设置的太小会导致频繁的CMS GC，产生大量的停顿，反之预留空间太小产生的新对象超过清理速度就会影起Concurrent  Mode Failure错误，虚拟机就会使用SerialOld收集器重新对老年代进行垃圾回收.如此一来，停顿时间变得更长。（其实CMS有动态检查机制。CMS会根据历史记录，预测老年代还需要多久填满及进行一次回收所需要的时间。在老年代空间用完之前，CMS可以根据自己的预测自动执行垃圾回收。这个特性可以使用参数UseCMSInitiatingOccupancyOnly来关闭。）
    - 并发问题：CMS默认的回收线程数是(CPU个数+3)/4，如果核数过少就让GC占用CPU过多，虽然有增量模式让允许GC标记和清理时候和用户线程交替运行，但是得不偿失
    - 内存碎片：因为CMS 是标记-清除算法很有内存碎片产生，当对象进入old没有足够的连续的内存导致直接FULL GC，CMS的解决方案是使用UseCMSCompactAtFullCollection参数(默认开启)，在顶不住要进行FullGC时开启内存碎片整理。这个过程需要STW，碎片问题解决了,但停顿时间又变长了，另外一个参数CMSFullGCsBeforeCompaction，用于设置执行多少次不压缩的FullGC后，跟着来一次带压缩的（默认为0，每次进入Full GC时都进行碎片整理）
    - cardTable 这个是为了标记 年老代对象引用对象概况的数据结构，一个byte数组，作用是为了减少YGC时候扫描老代年是否引用年轻代时的开销，基于卡表（CardTable）的设计，通常将堆空间划分为一系列2次幂大小的卡页（Card Page）,HotSpot JVM的卡页（Card Page）大小为512字节，卡表（Card Table）被实现为一个简单的字节数组，即卡表的每个标记项为1个字节,当一个对象引用改变即对象引用进行写操作，将会把8位中的标记项的引导为设置为dirty，带来了2个问题：多线程读写cardTable时出现，虚共享，即CPU在写的时候会把内容写进缓存行，造成性能开销，解决办法是开启JVM参数-XX:+UseCondCardMark，在执行写屏障之前，先简单的做一下判断。如果卡页已被标识过，则不再进行标识；还有问题是一个card的大小是512，而每个线程默认每次扫描数是256，每个stride就是512*256 =128K，这个大小对于现在计算机太小了，线程调度分配成本就会变大，此时可以通过ParGCCardsPerStrideChunk把每次收集的即一个stride的数量调大来增加效率这个数量要根据实际的业务场景进行调优,网上一般流传3个魔术数字:32768、4K和8K，这个值不能设置的太大，因为GC线程需要扫描这个stride中老年代对象持有的新生代对象的引用，如果只有少量引用新生代的对象那就导致浪费了很多时间在根本不需要扫描的对象上。


