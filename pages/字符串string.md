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
- replaceFirst，replaceAll，replace区别
	- replace(CharSequence target, CharSequence replacement) ，用replacement替换所有的target，两个参数都是字符串。
	- replaceAll(String regex, String replacement) ，用replacement替换所有的regex匹配项，regex很明显是个正则表达式，replacement是字符串。
	- replaceFirst(String regex, String replacement) ，基本和replaceAll相同，区别是只替换第一个匹配项。
- string对“+”的重载
	- java不支持运算符重载，其实只是java提供的语法糖
	- 重载：对已有的运算符重新进行定义，赋予另外一种功能
	- 语法糖: **指计算机语言中添加的某种语法，这种语法对语言的功能没有影响，但是更方便程序员使用。语法糖让程序更加简洁，有更高的可读性。**
	- 用+号的时候，内部是定义了个stringBuilder，用append方法进行处理的
	- 特殊情况：
		- 两个固定的字面量拼接，会进行常量折叠（两个字面量都在编译期可知），直接变成string s = "ab";
- 字符串拼接的几种方式和区别
	- 用+拼接字符串
		- 不建议在循环里用+拼接字符串，因为每次循环都会new出来一个StringBuilder对象，然后进行append操作，最后通过toString方法返回String对象，造成内存资源浪费
	- 用concat来拼接字符串
		- 创建一个字符数组，长度是已有字符串和待拼接字符串的长度之和，把两个字符串的值复制到新的字符数组中，使用这个字符数组创建一个新的string对象并返回
	- stringBuffer.append()
		- 和StringBuilder差不多，但是他的append方法用synchronized来修饰过的
		- 并发场景里要用stringbuffer
	- stringBuilder.append()
		- char[] value
		- 字符数组中不一定所有位置都被使用，所以有count来记录数组中已经使用的字符个数
	- StringUtils.join("a",",","b")
		- 将数组或集合以某拼接符拼接到一起形成新的字符串！！！
			- ```
			  String []list  ={"Hollis","每日更新Java相关技术文章"};
			  String result= StringUtils.join(list,",");
			  System.out.println(result);
			  //结果：Hollis,每日更新Java相关技术文章
			  ```
			- 其实内部也是StringBuilder
	- 效率对比
		- 用时对比`StringBuilder`<`StringBuffer`<`concat`<`+`<`StringUtils.join`
- java8中的StringJoiner
	- StringJoiner是java.util包中的一个类，用于构造一个由分隔符分隔的字符序列（可选），并且可以从提供的前缀开始并以提供的后缀结尾。虽然这也可以在StringBuilder类的帮助下在每个字符串之后附加分隔符，但是这个比较简单
	- ```
	  StringJoiner sj1 = new StringJoiner(":","[","]");
	  sj1.add("Hollis").add("hollischuang").add("Java干货");
	  System.out.println(sj1.toString());
	  [Hollis:hollischuang:Java干货]
	  ```
	- 内部还是用StringBuilder实现的
	- list.stream().collect(Collectors.joining(":"))
	- 日常使用推荐
		- 1、如果只是简单的字符串拼接，考虑直接使用"+"即可。
		- 2、如果是在for循环中进行字符串拼接，考虑使用`StringBuilder`和`StringBuffer`。
		- 3、如果是通过一个`List`进行字符串拼接，则考虑使用`StringJoiner`。
- String.valueOf和Integer.toString的区别
	- 没有区别，String.valueOf内部就是调用Integer.toString来实现的
- swith对String的支持
	- 目前支持，byte，short，int，char，string
	- 对整型支持的实现
		- 对int的判断是直接比较整数的值
	- 对字符型支持的实现
		- 对char类型比较，实际上比较的ascii码，会把char型变量转换成对应的int型变量
	- 对字符串支持的实现
		- 通过equals和hashCode方法来实现的
		- switch只能用整型，int
		- 因为hashCode也可能会发生哈希碰撞，还需要用equals再进行安全检查
- 字符串池
	- ```
	  String str = "Hollis"; 字面量
	  String str = new String("Hollis")；
	  ```
	- 当通过字面量方式创建字符串对象的时候，JVM会先对字符串进行检查，如果字符串常量池里存在相同内容的字符串对象的引用，就把引用返回
	- 否则就创建新的字符串对象，将引用放到字符串常量池，返回该引用
	- 常量池的位置
		- 1.7以前，是在永久代
		- 7的时候，从永久代移出，暂时放在堆内存里
		- 8的时候，从堆内存里放到永久代
- Class常量池
	- 三种常量池
		- 字符串常量池
		- Class常量池
		- 运行时常量池
	- 什么是Class文件
		- **如何使用16进制打开class文件：使用 **`vim test.class`** ，然后在交互模式下，输入**`:%!xxd`** 即可。**
		- 字节码文件，Class文件里包含了java虚拟机指令集和符号表一集若干其他辅助信息
		- 版本号后面，就是Class常量池入口了
		- Class文件中除了类的版本，字段，方法，接口等描述信息外，还有一项信息就是常量池，用来存放编译器生成的各种字面量和符号引用
		- ![-w697](http://www.hollischuang.com/wp-content/uploads/2018/10/15401192359009.jpg)
		- 常量池计数器是从1开始的
		- 也可以通过javap -v HelloWorld.class
	- class常量池有什么
		- 字面量
			- 字面量就是指由字母、数字等构成的字符串或者数值。
			- 字面量只可以右值出现，就是等号右边的值
		- 符号引用，相对的是直接引用
			- 类和接口的全限定名
			- 字段的名称和描述符
			- 方法的名称和描述符
	- Class常量池有什么用
		- Class常量池是Class文件中的资源仓库，其中保存了各种常量。而这些常量都是开发者定义出来，需要在程序的运行期使用的。
		- Class是用来保存常量的一个媒介场所，并且是一个中间场所。在JVM真的运行时，需要把常量池中的常量加载到内存中。
-
-
-
-
- TODO 写26岁感言
- TODO 列出来三四月份的学习计划
- TODO 换枕头套
- TODO 护肤化妆
-