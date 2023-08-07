# ARMv8-A编程指导之Caches

## 1 Cache术语

冯诺依曼架构中，单个cache用于指令和数据（统一cache）

哈佛架构有分开的指令和数据总线，因此有两个cache：指令cache（I-cache）和数据cache（D-cache）

![img](https://cdn.jsdelivr.net/gh/sym1121/pictures/202307210935556.png)

![img](https://cdn.jsdelivr.net/gh/sym1121/pictures/202307210934595.png)

① tag为存储在cache中的内存地址的一部分，它来区分与数据线相关的主内存地址

②逻辑块为cache line，它为最小的cache可加载单元，内存中连续的一块字。当它包含可缓存的数据或指令时cache line被认为valid，否则被认为invalid。

③index为内存地址的一部分，它决定地址可以在哪个cache line中被找到。

④way为cache的细分，每种方式的大小相等，并以相同的方式索引。Set由共享某个index的所有way的cache line组成。

⑤这意味着地址的最后几位，称为offset，并不要求被存储在tag。

### 1.1 组相连cache和路

 ARM核的cache通常使用一组相连的cache实现。增加cache的相连减少了thrashing的可能性。最理想的为全相连cache

如下图为2路cache。来自地址0x00，0x40或0x80的数据可能在line0中被找到，但并不是在两个cache路中同时被找到。

![img](https://cdn.jsdelivr.net/gh/sym1121/pictures/202307210937560.png)

### 1.2.Cache tags和物理地址

①每个cache line都有一个tag与之相关，该tag记录了与cache line相关的外部内存的物理地址

②Cache中每个cache line包含：

1. 相关物理地址的tag值；
2. 有效位用来表明该line是否在cache中，即tag是否有效。如果cache的一致性跨越多个core时有效位也可以为MESI的状态位；
3. Dirty数据位表明是否cache line中的数据和外部内存中的数据保持一致；

 ③一个组相连的cache明显减少cache thrashing的可能性且改善程序执行速度，但代价为增加硬件复杂度和对功耗有轻微增加。

​    一个简单的4路组相连的32KB L1 cache（比如[Cortex](https://so.csdn.net/so/search?q=Cortex&spm=1001.2101.3001.7020)-A57处理器的数据cache），16word cache line长度，如下图所示：

![img](https://cdn.jsdelivr.net/gh/sym1121/pictures/202307210941806.png)

### 1.3 包含和独占caches

考虑一个简单的内存读，比如在单个core处理器上执行LDR X0, [X1]。

（1）如果X1指向内存中的一个位置，它被标记为可cacheable，然后再L1数据cache中会有cache查找；

（2）如果在L1 cache中找到地址，然后数据从L1 cache中读取并返回给core；


![img](https://cdn.jsdelivr.net/gh/sym1121/pictures/202307210951086.png)

（3）如果在L1 cache中没有找到地址，但在L2 cache中，cache line会从L2 cache加载到L1 cache中，然后数据返回到core。这会导致cache line从L1中被回收腾出空间，但仍在L2 cache中存在；



 （4）如果地址既不在L1 cache也不再L2 cache中，数据被同时从外部内存加载到L1和L2 cache中，然后返回给core。这也会导致<u>**cache line被回收**</u>。


![img](https://cdn.jsdelivr.net/gh/sym1121/pictures/202307210954410.png)

### 2 cache控制器

Cache控制器为负责管理cache内存的硬件模块，它是以一种程序不可见的方式。它主动将代码或数据写到cache，从core中获取读写内存请求。当它接受core的请求，会检查地址是否在cache中——cache查找过程。 当core请求某个地址的指令或数据时，在cache tag中没有匹配，或tag无效，cache没有命中，请求必须被传递到内存层次的下一级，如L2 cache或外部内存。

### 3 cache策略

 	 当一个cache line被分配为数据cache，cache策略描述了当store指令执行且在数据cache中被命中时

​		cache分配策略如下：

***\*Write allocation（WA）:\****

​	 	一个cache line在写未命中时分配。这意味着在处理器上执行store指令可能会导致突发读产生。对于cache line，在写发出前通过linefill来获取数据。cache包含整个cache line，它是加载的最小单元，即使你仅在cache line写单个byte。

***\*Read allocation（RA）\****

​	 	一个cache line在读未命中时分配。

  	  cache更新策略如下：

***\*Write-back（WB）:\****

​		写只更新cache且将cache line标识为脏。当cache line被回收或明确的清除时才更新外部的内存。

![img](https://cdn.jsdelivr.net/gh/sym1121/pictures/202307211025761.png)

***\*Write-through（WT）\****

​		写同时更新cache和外部内存系统，这不会标记cache line为脏

![img](https://img-blog.csdnimg.cn/79602ac50319457dad8ed305a1d7e562.png)

​		数据读在cache中命中的行为在WT和WB cache模式中一样。

正常内存的[cacheable](https://so.csdn.net/so/search?q=cacheable&spm=1001.2101.3001.7020)属性可分为inner和outer属性。Inner和outer的分界线由实现决定的。通常，inner属性用于集成cache，outer属性用于外部cache使用的处理器内存总线。

![img](https://cdn.jsdelivr.net/gh/sym1121/pictures/202307211029967.png)

### **4 PoC和PoU**

 对于组为基础和路为基础的清除和无效化，操作针对的是cache中的某一级cache。对于使用VA的操作，架构定义了两个点：

（1）一致性点PoC（Point of Coherency）

对于某个地址，PoC为所有能够访问内存的观察者比如core，DSP，dma引擎保证看到一个内存的位置是相同内容。通常，主要是外部系统内存。

![img](https://cdn.jsdelivr.net/gh/sym1121/pictures/202307211039559.png)

（2）统一点PoU（Point of Unification）

对于core来说，PoU为指令/数据cache以及core中转换表walk保证看到一个内存位置是相同的内容。比如，一个统一的L2 cache将是哈佛系统中L1 cache和缓存转换表项的[TLB](https://so.csdn.net/so/search?q=TLB&spm=1001.2101.3001.7020)的PoU。如果没有外部cache存在，主存为PoU。

![img](https://cdn.jsdelivr.net/gh/sym1121/pictures/202307211039037.png)

### 5 cache维护

有时软件进行清除或无效化cache是必要的。

​	①cache或cache line的无效化意味着通过清cache line中的valid位来清楚cache line中的内容。在复位之后cache必须为无效的因为它的内容没有定义。这也可以看着是使cache之外的内存域的更改对cache用户可见的一种方式。

​	②清处ache line意味着写cache line的内容将其标为脏，写入到下一级cache，或到主存，并在cache line中清脏页。这使得cache line的内容与下一级的cache或内存系统中一致。这仅应用于在回写策略使用时的数据cache。这也是一种在cache中修改，让outer内存domain的用户可见，但这仅对数据cache有用。

​	③清零。这会将cache中的内存块清零，而不需要首先从outer domain读取它们的内容。这也仅对数据cache有用。

### **6 cache的发现**

cache维护操作可以从过cache set或way或VA来发出。平台无关的代码需要知道cache的大小，cache line的大小，组和路的数目，有多少级的cache。此需求最可能出现在post-reset后cache无效化和清零操作中。所有其他对架构cache的操作都是基于PoC和PoU。






