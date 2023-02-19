- 能根据异常信息迅速知道哪个区域的内存溢出，知道怎样的代码可能会导致区域内存溢出，出现异常后如何处理
- java堆溢出
	- java堆用于存储对象实例，不断创建对象，保证GC ROOTS到对象之间有可达路径来避免垃圾回收机制清除这些对象，随着对象数量的增加，总容量触及最大堆的容量限制后就会产生内存溢出异常。
	- 配置
		- ```
		  VM Args：-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
		  ```
		- HeapDumpOnOutOfMemoryError 让虚拟机在出现内存溢出异常的时候dump出当前的内存堆转储快照进行分析
		- 排查思路
			- 确认内存中导致OOM对象是否是必要的，要分清楚出现了内存泄露memory leak还是内存溢出 memory overflow
			- 内存泄露 通过工具查看泄露对象到GC Roots引用链，找到泄露对象是通过怎样的引用路径，与哪些GC Roots相关联，导致垃圾收集器无法回收，根据泄露对象的信息以及到GC Roots引用链的信息，可以准确定位到对象创建的位置，找到产生内存泄露的代码的具体位置。
			- 不是内存泄露，检查java虚拟机的堆参数和机器内存对比，是否有向上调整的空间。从代码检查是否存在对象声明周期过长，持有状态时间过长，存储结构设计不合理的情况。
- 虚拟机栈和本地方法栈溢出
	- 对于HotSpot来说，-Xoss参数（设置 本地方法栈大小）虽然存在，但实际上是没有任何效果的，栈容量只能由-Xss参数来设定。
	- hotspot虚拟机不支持栈的动态扩展，除非是创建线程申请内存的时候无法获得足够内存导致OOM,否则只会无法容纳新的栈帧导致StackOverflowError
	- 无论是由于栈帧太大还是虚拟机栈容量太小，当新的栈帧内存无法分配的时候， HotSpot虚拟机抛出的都是StackOverflowError异常
	- 出现StackOverflowError异常时，会有明确错误堆栈可供分析，相对而言比较容易定位到问题所 在。
	- “unable to create native thread”后面，虚拟机会特别注明原因可能是“possibly out of memory or process/resource limits reached”。可能代表建立过多线程导致内存溢出。
- 方法区和运行时常量池溢出
	-