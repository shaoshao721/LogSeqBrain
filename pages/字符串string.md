tags:: string，java

- 不可变对象是在完全创建后其内部状态保持不变的对象
- 一个string对象在堆中被创建出来，就无法被修改
- 为啥要设计成不可变
	- 缓存
		- 字符串用的很多，大量创建字符串非常耗费资源
		- 字符串池
			- 两个内容相同的字符串变量，在池中指向同一个字符串对象，节省内存资源
		- 能用字符串池，就是因为字符串不可变。如果可变的话，其中一个改变了，另外用这个字符串的地方也会变了
	- 安全性
		- 字符串存储了很多敏感星系，jvm类加载器在加载类的时候也广泛使用了
		- 如果可变的话，字符串就可能被修改，那字符串内容就不能相信了
	- 线程安全
		- 不可变对象在多个同时运行的多个线程里共享
		- hashcode缓存
			- 字符串的值不会改变，hashCode在string类中被重写，以方便缓存。第一个hashCode调用的时候计算和缓存散列，从那时起返回相同的值
- JDK6和JDK7中substring的原理及区别
	- substring() substring(int beginIndex, int endIndex)
	- 截取字符串返回beginIndex，endIndex-1范围内的内容
	- JDK6
		- string是字符数组实现。jdk6里，char value[]，int offset, int count.存储真正的字符数组，数组第一个位置索引，字符串中包含的字符个数
		- 调用substring方法的时候，会创建一个新的string对象，string的值仍然指向堆中的同一个字符数组，改变count和offset的值
		- ![string-substring-jdk6](http://www.programcreek.com/wp-content/uploads/2013/09/string-substring-jdk6-650x389.jpeg)
		- 问题
			- 有一个很长很长的字符串，切割了之后只要很短的一段。但是因为这边还在引用堆里的那个字符数组，所以这个长字符串一直没法回收，导致及内存泄露
			- java6用这种方式解决，生成新的字符串引用他
			- ```
			  x = x.substring(x, y) + ""
			  ```
	- JDK7
		- 会创建个新的数组
		- ![string-substring-jdk7](http://www.programcreek.com/wp-content/uploads/2013/09/string-substring-jdk71-650x389.jpeg)
		-