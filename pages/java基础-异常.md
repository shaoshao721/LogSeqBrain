tags:: 异常，java

- try-catch-finally的执行顺序
	- ```
	  try {                        
	      //执行程序代码，可能会出现异常                 
	  } catch(Exception e) {   
	      //捕获异常并处理   
	  } finally {
	      //必执行的代码
	  }
	  ```
	- 执行顺序
		- try 如果没有捕获到异常，try块里的内容一一执行，程序跳过catch，执行finally语句及其后的语句
		- try有异常，catch语句里没有捕获这个异常的情况。异常会抛给JVM处理，finally语句块里的语句还是会被执行，finally语句之后的语句不会被执行
		- try有异常，catch里也有对应的处理。try里按照顺序执行，执行到异常之后，跳到catch语句块里，找到对应的catch块，catch块里执行完之后，如果有return，暂存return的结果，继续执行finally语句。最后返回到catch里return
- try-finally
	- 一般finally里放一些代码用来关闭资源，关闭连接。
	- finally不会执行的情况
		- 用了system.exit()退出程序
		- finally块里也发生了异常
		- 断电了
		- 关闭CPU
		- 程序所在的线程死了
- try-with-resource
	- 更优雅的方式来实现资源的自动释放，自动释放的资源需要是实现了 AutoCloseable 接口的类。
	- ![image.png](../assets/image_1678608505202_0.png)
	- try代码块退出的时候，自动调用Scanner.close方法。
	- 如果是原来的，把scanner.close放在finally代码块的话，在finally里抛出异常，不会继续走。现在这种，会被抑制，抛出的还是原始异常，
	- 被抑制的异常会由 addSusppressed 方法添加到原来的异常，如果想要获取被抑制的异常列表，可以调用 getSuppressed 方法来获取。
- 异常实践指南
	- 只针对不正常的情况才使用异常。
		- Java 类库中定义的可以通过预检查方式规避的RuntimeException异常不应该通过catch 的方式来处理，比如：NullPointerException，IndexOutOfBoundsException等等。
			- 异常机制的设计初衷是用于不正常的情况，所以，创建，抛出，捕获异常的开销很昂贵
			- 把代码放在try-catch里阻止了JVM实现本来可能要执行的某些特定的优化
			- 对数组进行遍历的标准模式并不会导致冗余的检查，有些现代的JVM实现会将它们优化掉。
	- ### 在 finally 块中清理资源或者使用 try-with-resource 语句
		- 在try块的最后释放资源，会导致在中间抛出异常之后，释放资源的代码就被跳过去了
		-