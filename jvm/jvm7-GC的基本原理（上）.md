#jvm7 GC的基本原理（上）

> 垃圾回收算法手册----自动内存管理的艺术

**GC并不仅仅是对象的回收，分配和回收是一个有机的整体**

**GC并不仅仅是对象的回收，分配和回收是一个有机的整体**

**GC并不仅仅是对象的回收，分配和回收是一个有机的整体**

------

## 虚拟机为什么需要GC能力

现代的语言都允许在运行期间能动态的分配内存。那么分配出去的内存自然要有一个回收的机制来确保不用的内存能及时回收，不然就会有内存泄漏等问题。如果内存的分配和释放都由我们自己编写的代码来控制，那么就会有正在使用的内存被误释放、不用的内存始终得不到释放等问题，这样的问题难以定位难以重现。所以一个现代化的语言都会提供自动内存管理的能力来帮助管理内存的占用。

## 三色抽象

利用三色抽象可以推演回收器的正确性。回收器将对象图划分为三种颜色，黑色（确定存活）、白色（可能死亡）。任意对象在初始状态下都是白色，当回收器初次扫描到某一对象时将其着为灰色，当完成该对象的扫描并找到其所有的子节点之后，回收器将会将其着为黑色。
在整个垃圾回收的过程中，所有标记到的对象都是灰色，所有标记并遍历完的对象都是黑色，所有未被遍历到的对象是白色。因此三色抽象是一个单向的颜色转换：**白色<span>&rarr;</span>灰色<span>&rarr;</span>黑色**。

## 对象填充

由于程序运行的过程中会分配各种各样的对象，那么对象的大小也有各种各样的可能性。对象大小的不同会对内存分配和回收造成很大的麻烦，其中比较典型的就是内存碎片问题。所以现在一般的做法就是通过对象填充的方式，把一些小对象通过填充无用字节的方式进行对齐，这样可以极大的提高内存的分配和回收的效率。

一个Java对象的大小如何计算？

虚拟机规范中并没有规定一个Java对象应该占用多少字节的内存，所以这块内容是虚拟机实现自由控制的。那么以hotsport（openjdk）为例，一个Java对象到底占用了多少内存。

```java
Object obj = new Oject();
// 复习一下oop.hpp的内容
// 由于只是new了一个对象，所以这个对象的空间占用仅仅是对象头的这部分开销
// 对象头分为两个部分：mark word和指针
// 其中mark word占用8个字节（64位8个字节，32位4个字节，如果开启了对象压缩，那么64位机器也是4个字节）
// 指针占用8个字节
// 所以一个Java对象在64位机器上占用16个字节（不考虑指针压缩）
```

> 这也是大家能看到为什么会说包装类（Integer、Long等）内存空间占用不高，一个int占用4个字节，一个Integer则是一个对象16字节+内部的实例字段（在讲oop的时候有说过一个oop的组成就是对象头+实例字段，如果总大小不是8的倍数，就通过对象填充补齐到8的倍数）
>
> **复习：**顺便回顾一下实例字段的分配：先分配父类（如果父类还有父类就递归向上），然后逐个分配。子类会包含所有父类的实例字段（不论是private protected public），实例字段的分配顺序是可以配置的，默认顺序见之前的图，对象指针在最后。

##惰性清扫

惰性清扫顾名思义就是等到需要用的时候再去回收垃圾，惰性清扫的逻辑一般是如下模式

* stop the world。从gc roots开始扫描，扫描的过程中，标记哪些是垃圾，那些是可用内存块，垃圾不急于回收，放入该内存块对应的回收队列（linked list）中。
* 分配器正常分配内存，如果分配失败，则执行惰性清扫，在对应大小的内存块中执行垃圾回收，直到获取了足够的内存空间。
* 如果内存块中无法分配出足够多的内存，那么就申请一块新的内存分配。

其实惰性清扫我觉得就可以把他当作WeakReference，当需要用内存的时候再把垃圾都回收。

> 在看过Redis的源码之后，Redis的内存回收基本上属于这样的惰性回收的策略，惰性+定时。

###惰性清扫的特点

1. 程序的局部性比较好。因为我去回收的这块内存是我回收了立即就要用的，所以各类高速缓存（cache line等）命中缓存的概率会非常的高。
2. 如果堆中的大部分对象都是空闲的，惰性清扫的性能非常卓越。

##标记清扫算法

算法本身我们就快速过一下，标记清扫就是通过一个标记环节分辨出哪些是可回收垃圾，然后再通过一次清扫过程把所有标记到的垃圾都清除。

###标记清扫的典型特点

- 分配和读取内存时不会有额外的开销

  在读写内存的时候通过指针就可以直接找到该对象，没有太多的额外开销。

- 吞吐量

  使用惰性清扫的标记清扫算法，往往拥有较高的吞吐量。标记阶段往往是指针追踪的开销（就是从GC Roots找到所有的垃圾，这个过程是通过追踪对象之间的指针来完成的），不论是或者不是垃圾，对于对象的操作往往也只需要设置一个标记位或者标记字节。虽然在标记阶段也需要挂起所有线程，但是总体开销还是比较小的。

- 空间利用率
  标记清扫拥有更高的空间利用率。他的标记位的空间不大，可以通过对象头的一个标记位或者单独的位图来完成

  引用计数需要一个对象槽来记录计数，复制式则只能使用1/2的空间。

- 非移动式算法
  非移动式的算法优点和缺点并存，不移动对象，则可以与编译器和回收器不合作的场景。即我管我自己回收就可以。存在的问题就是会逐渐碎片化。

  问题就是碎片过多，无法分配更大的对象，但是可以根据内存对象，成簇出现，成批死亡的特点，可以减轻碎片化的管理（比如同一大小的对象分配到同一个内存块中）。

##标记整理算法

标记整理算法需要经过2个阶段，

1. 标记阶段。标记哪些是垃圾哪些不是，通常会伴随STW。
2. 整理阶段，即移动存活的对象，同时更新存活对象中所有指向被移动对象的指针。

但是要注意，标记整理的不同实现算法堆的遍历次数、整理过程以及对象迁移的方式都有不同，这些不同的实现方式会对性能有不同的影响。

### 整理的顺序

- 任意顺序：对象的任意移动他们的顺序，和原始排列没有关系

- 线性顺序：将具有关联关系的对象排列在一起，比如下面这个例子，A和B就可以整理到一起

  ```java
  public class A {
    private B b;
  }
  
  public class B {
    private String b;
  }
  ```

- 滑动顺序：将对象滑动到堆的一端，“挤出”垃圾，从而保持对象在堆中的原有分配顺序。这个大家可以想象挤牙膏的方式，朝着一侧挤压，如果是垃圾就挤出去（回收掉），然后后面的存活对象就被挤过来填充这个位置。

**现代标记-整理回收器均使用滑动顺序**。因为这个算法不会改变对象之间的相对顺序，也就是赋值器的局部性会非常的好（能充分利用高速缓存）。而任意顺序和线性顺序由于会打乱原先的内存分布，所以会大幅度降低应用程序的吞吐量。

现代整理算法：

1. 双指针整理算法：该种方法是任意顺序整理，虽然算法简单执行高效，但是打乱了堆中对象的布局
2. Lisp 2 算法：这是一种滑动回收算法，需要在对象头部一个额外的槽来保存转发地址
3. 引线整理算法：该算法不需要额外的空间开销，但是需要两次堆便利，每次遍历的开销都很高。
4. **单次遍历算法：这是一种现代的滑动回收算法，不需要额外的空间开销，转发地址可以实时算出。**

### 考虑的问题

- **整理的必要性：**在一个长时间运行的系统中，内存的碎片几乎是必然的事情。所以标记清除算法通常适用于对象大小通常差不多的场景（可以考虑对象之间使用对齐的策略）。**在一个java web应用中，整理是必要的。**
- **整理的吞吐量开销：**整理式堆中分配内存的速度会非常快（因为整理之后，内存基本上都是顺序分配）。而且相比于复制式算法，其内存需求只有他的一半。但是整理算法通常吞吐量比较低（除了单次遍历算法）。**所以一个通常的解决方案就是尽可能多的使用标记清除算法，直到有必要的时候，才使用标记整理算法。**
- **长寿数据：**在一个长时间运行的应用中，长寿对象，甚至是永生对象的出现，并且堆积在堆底也是很常见的。复制式收集器针对这样类型的数据是表现很差（因为每次都需要机械的在两个区域之间来回复制）。所以可以采用分代收集。但是针对最老的一代进行回收时还是需要处理这些长寿对象。但是标记整理可以把标记到的这些长寿对象忽略即可。

##复制式回收

标记清扫的开销比较低，但是有内存碎片的问题；标记整理算法虽然本身开销比较大，但是赋值器的性能得到了提升；还有一种算法，只需要一次堆遍历，也可以同时完成整理的工作，这就是复制算法，但是复制算法的可用空间降低了一半。

### 半区复制回收

半区复制就是把堆划分为两个相等的半区（来源空间和目标空间）。而且通常堆也不一定需要时连续的堆。算法也很简单，在一个半区中简单的增加空闲指针，如果空间不足，就互换角色，把这个半区中的存活对象堆到另一个半区的一端。然后丢弃该空间。这样存活对象密布在一个半区。

###经典算法

**传统半区复制：**找到GC Roots并遍历，把GC Roots对象A，从From复制到To，并且加入队列中，然后遍历A持有的对象，直到所有存活对象都遍历结束。这个时候把队列中的对象指针都指向复制后的对象。然后清空From空间和队列。

**Cheney Scanning：**使用对象的广度优先遍历，同时去除了经典算法中的队列，利用灰色对象（三色抽象）以及一个额外的指针即可实现先进先出队列。

###复制算法的特点

- 空间利用率比较低。因为要一分为二，一个From一个To。
- 优秀的局部性。由于复制算法使得对象是顺序分配，提升了赋值器的局部性，从而提升了各个层次的缓存命中率。
- 复制算法中的Cheney Scanning算法是一个广度优先的算法，会使父子节点分离，降低程序的局部性。

### 需要考虑的问题

***分配：***由于在经过整理的堆中分配对象其过程非常的简单，通常只要判断堆或者内存块的上限，就可以分配空闲指针。而且通常来说，顺序分配高速缓存的命中率会更高。

***空间与局部性：***通常由于半区复制算法，会将整个堆划分为二，这到底是会导致性能的提升，还是性能的下降，还是取决于赋值器与分配器之间的平衡、应用程序的的特点以及可用堆的大小空间。通过简单的分析可知，更大的堆用复制式会更好。而且虽然复制式算法使用顺序分配，提高了高速缓存的命中率，但是由于复制式的移动对象，会将堆中的对象重排列，所以也会引起性能的下降。

***移动对象：***由于非移动式回收器不用去更新所有指向某一个对象的引用，但是移动式回收都需要，而且并发式回收还需要确保多线程下原子的更新引用。同时，由于对一个对象的复制开销，是比较大的，如果非常频繁的复制一些大对象也会导致回收器的性能下降。

##引用计数

引用计数其实非常简单，而且作为一种基本的内存管理回收算法，一直保留到现在。通常引用计数的数字可以保留在对象的头部。每当回收一个节点，都会引起子节点的递归回收。

###引用计数的特点

#### 优点

1. 当一个对象成为垃圾后会得到立即回收，提高程序的局部性。
2. 由于是一个立即回收的算法，不像其他追踪式回收需要预留一定的空间（比如内存占用到70%开始执行一次回收）。
3. 算法实现非常简单，机制易于理解。

####缺点

1. 循环遍历时开销。比如一个list，list中的对象引用计数频繁的加一减一，因为for循环等时代码中非常常见的操作，所以这个开销会大很多。
2. 并发情况下的计数安全问题。需要cas去修改程序的计数。
3. 循环依赖问题。
4. 空利用率低。如果一个对象本身很小，但是被大量的引用，就会造成空间利用率上的浪费。比如一个Integer.ZERO对象，这个对象本身的有效信息仅仅是一个4字节的int，但是如果这个对象被堆内数量非常庞大的对象引用，那么存计数的这个字段就会远远超过有效信息。
5. 回收时的卡顿。由于引用计数要求计数降为零就立即回收，那么如果该对象有很多子节点，那么就会在遍历回收的过程中造成程序卡顿。

> 这里列举的都是我自己的看法，不一定完全正确。
>
> 1. try catch finally的引用计数统计比较困难。比如我一个方法爆了一个空指针，那么当前线程一直到被catch这个异常之前，所有分配到的对象都需要引用计数-1，这个开销也非常大。
> 2. 引用计数难以实现除强引用之外的引用类型。比如Java支持强软弱虚四种引用类型，引用计数要支持软、弱引用比较困难（虚引用放进ReferenceQueue）

###解决上述缺点的思路

- 惰性回收。如果是垃圾就不要立即回收，而是放到回收队列中当需要的时候再回收。
- 合并操作。比如循环中，在循环开始时记一次，结束时再记一次，就不用所有的对象频繁的记录。
- 循坏依赖的解决：
  - 在循坏依赖中，可以通过试用删除的做法，每次认为其中一部分对象是垃圾，进行使用删除，如果没有对象的引用计数变为0，就可以知道没有循坏依赖。
  - 针对引用计数不为0的对象，使用追踪式回收器回收，引用计数回收器回收能回收的垃圾。
- 受限域计数。每个对象的计数有一个上限，超过上限的对象使用追踪式回收器来回收。

### 需要考虑的问题

引用计数回收实效性非常好，也带来了非常好的局部性，也不需要再回收时遍历整个堆。

但是该算法带来的问题也很多，同时每一种针对策略都只会带来更大的复杂性，和引用计数本身高效简单的特点不匹配。

引用计数把内存的分配、内存的管理、垃圾的回收变成了一个连贯的过程，而不像其他回收器是可以被解耦的。

## 内存分配

### 顺序分配

顺序分配就是一开始申请一整块内存，然后从从内存块的一端开始分配，数据结构简单，只需要一个表示界限的指针和一个空闲内存的指针就可以实现顺序分配，顺序分配就是简单的移动空闲内存指针即可

###特点

简单高效、程序的局部性更好。缺点是顺序分配和非移动式回收算法不是很匹配，因为会造成很多的内存碎片。

###空闲链表分配

空闲链表分配就是通过一个空闲链表来记录空闲的内存单元位置和大小，数据结构不一定要使用链表，只不过通常都用链表实现。

空闲链表分配就是根据链表上记录的空闲内存块，在链表上找到一个可以使用的空闲内存块使用即可。

寻找可以使用的空闲内存块被称为**顺序适应分配**，主要有三种算法：

#### 首次适应分配

即从链表上找到第一个合适大小的内存块就分配。如果找到的内存快大于所需要的内粗，就将该内存块分解。多余的空间归还到空闲链表中。

- 内存单元会从小到大排列，链表头部会集中大量较小的空闲内存碎片。

  因为每次链表都是从head节点开始遍历，只要找到一个合适大小的就分配，那么小对象总是在链表头部找到合适的内存块，因此链表头部会集中大量碎片，时间长了之后会降低分配内存的效率

#### 循环首次适应分配

这个就是上面算法的变种，不要每次都从链表头节点，而是从上一次找到的节点开始。

这个算法看上去很好，但是实际上有很多问题。因为链表中相邻节点的存活对象不是同一时刻分配的（链表遍历结束会从头开始），所以这些对象被回收的时间点也不尽相同，这样反而会加剧内存的碎片化，同时同一批次的对象有可能散落在各个地方，从而降低程序的局部性。

####最佳适应分配

这个就是遍历整个链表，找到最最合适的那一个内存块来分配。这个方式的优点就是内存浪费率比较低，但是性能会比较差。

### 空闲链表加速分配

如果堆比较大，单链表就会显得力不从心，比如用平衡二叉树来代替链表、不同的内存大小块使用不同的链表，避免在同一个链表上分配大小不同的对象等。

### 分区适应

上面说的不同大小对象使用不同的链表其实就是一种分区适应的思想。所谓分区适应本质就是寻找一个阙值。

使用内存分区处理的原因包括对象的移动性、大小、更低的空间消耗、更简单的对象性质识别、垃圾回收效率的改善、停顿时间的降低、更好的实现局部性等。（其实进行分区的目的就是在程序运行的过程中有各种不同大小不同特点的对象产生和消亡，针对不同类型的对象进行特殊的优化往往有利于提高整体的性能）

####分区策略

- **根据移动性进行分区。**比如典型的就是IO对象（比如Netty就使用堆外内存来管理这部分对象）
- **根据对象大小进行分区。**根据大小分区主要可以避免内存碎片化的问题。
- **为空间进行分区。**这个就有点jvm分代的感觉，根据对象分配频率、死亡比例等不同的特点，分配到不同的分区，使用不同的gc算法管理。
- **根据类别进行分区。**比如同一个Class的对象放在一起。这样做的好处可以提高程序局部性，因为一些共享字段可以放到一个共享区域等。
- **为效益进行分区。**垃圾回收策略都着眼于最有可能成为垃圾的对象上，来达到通过最少的努力来清理最多存储空间的目的。例如分代回收器就会频繁的对堆中某一个空间回收，回收器允许一些垃圾不被清理。因此垃圾回收器会比理想情况下调用的更加频繁。
- **根据线程分区。**使用线程本地子堆来进行分配（比如Java TLAB）。
- 其他。为局部性进行分区、根据可用性进行分区、根据易变性分区、为缩短停顿时间进行分区等（感觉没多大实践价值就忽略了）

## 分代回收

垃圾回收器的目的就是找到垃圾，在垃圾数量比较少的情况下，追踪式回收器，特别是复制式垃圾回收器，可以更高效的回收，但是长寿对象会影响效率。因为回收器会反复的针对这些对象操作。因此分代垃圾回收器时就是针对这些对象的一个提升，就是尽可能少的去处理这些长寿对象。***弱分代假说，表示，大部分对象都会在年轻时死亡，因此可以利用这些特性来提高回收的效益。***

通常在分代回收策略中，年轻代都会使用复制算法，通常情况下一次年轻代的回收不会超过10ms。

绝大部分分代回收器不会去扫描整个堆，而是仅仅扫描当前分代中的对象。所以分代间指针引用就会成为一个非常重要的问题。（如果一个年轻代对象被一个老年代对象所引用，那么这个对象非但不会被回收（因为老年代对象无法确定是不是垃圾，没有对整个堆进行扫描），而且活过了多次gc之后，该对象还会因为年龄增大，被提升到老年代之中）

### 分代与堆布局

分代可以使用物理上或逻辑上的分代，可以将分代限制在相同大小的空间内；内部的数据结构可以是扁平的，也可以是其他数据结构。

如果年轻代太小，则会因为回收太频繁，导致对象没有足够的时间死亡而回收的内存不多，同时频繁的垃圾回收也会增大停顿线程和扫描堆栈的开销。

其次，老年代会迅速的被年轻代填满，同时也会出现庇护的问题（就是老年代持有年轻代的引用而无法回收）。

###多分代

如果回收使用更多的分代，不仅可以快速回收年轻代，也可以降低老年代的填充度。

但是多分代最大的问题就是分代间指针，会给赋值器的写屏障带来很大的压力，同时也会带来很多算法的复杂度，因此现代的回收器一般都是使用两个分代。

### 存活对象的柔性提升

由于年轻代中有一半的空间要作为复制保留区，因而浪费率太高。所以就将年轻代划分为一个较大的诞生空间和两个较小的存活对象半区。比如HotSpot，ParNew的Eden : From : To = 8 : 1 : 1。

###自适应算法

由于GC是一个非常动态的过程，所以在此也需要一些自适应算法自动调整。比如程序的停顿时间、堆之间的比例等。

###分代间指针

在某个分代回收之前，回收器必须先确定该分代的根，除了寄存器、栈、全局变量等GC Roots之外，如果有一个分代a引用了分代b的对象，同时分代a不会参与本次回收，那么分代a也应作为本次回收的gc roots之一。因此分代间指针式需要的。分代间指针创建有三种方式，一种是创建对象时写入，二是赋值器更新指针槽时写入，三是将对象移动到其他分代是产生。只有记录分代间指针，才能确保对某一分代进行回收时GC Roots的完整性，有时，统称为回收相关指针。

#### 记忆集（Memory Set）

记忆集是一种记录分代间指针的数据结构，大致的意思是别人家的分代，哪些东西是指向我们这个分代的，我就把它记录下来。好处就是，如果我的某个对象被提升了，那么就可以根据记忆集来更新指针的来源；由于只记录指针的来源，那么即便当中修改过也没关系，因为在程序运行的过程中，会反复的写，所以通过这样一个指针，只需要在回收时根据记录的最终状态来处理就可以了。

### 需要考虑的问题

分代垃圾回收是一种高效的对象组织方式，可以显著提升性能。分代的有点主要体现在两个方面

1. 长寿对象的处理频率降低，而且使得中年对象有足够的时间到达死亡；
2. 分代回收器在分配新生对象时，通常可以使用顺序分配的策略（一般会使用复制算法），又由于大多数写操作都是针对年轻对象发生的，所以程序局部性也得到了进一步提升。

但是，分代垃圾回收如果一些程序中的对象年轻对象不具有较高的死亡率，则分代回收的策略可能就不再适用了（基本上都是适用的）。