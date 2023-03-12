tags:: 基础，泛型

- 为什么会引入泛型
	- 为了参数化类型，不创建新的类型的情况下，通过泛型指定的不同类型来控制形参具体限制的类型
	- 可以用在类，接口，方法中
	- 意义：
		- 适用多种数据类型执行相同的代码
		- 比如一个a+b的方法，需要适配多种类型，如int，double，float等，如果不用泛型，就要重载多个，很麻烦又没必要
		- 泛型
			- ```
			  private static <T extends Number> double add(T a, T b) {
			  	return a.doubleValue() + b.doubleValue();
			  }
			  ```
		- 泛型中的类型在使用的时候指定，不需要强制类型转换（类型安全，编译器会检查类型，因为是伪泛型吧
- 泛型基本使用
	- 使用方式:类，接口，方法
	- 类
		- 在类名后面加上泛型的标识符，泛型的标识符可以有多个
	- 接口
		- 也是在接口定义上加上泛型的标识
		- 子类在实现这个接口的时候，也要加上这个泛型的标识
	- 方法
		- 调用方法的时候指明泛型的具体类型
		- ```
		  public <T> T getObject(class<T> c) {
		  	T t = c.newInstance();
		      return t;
		  }
		  ```
		- 要在返回值定义的前面加上泛型的标识，来标明这个方法持有哪些泛型类型，同时声明这个方法是泛型方法
		- 参数中传入泛型T的具体类型，对象c用来创建泛型T代表的类的对象（用反射的方法来做的）
			- 因为泛型方法的话，意味着我们不知道具体的类型是啥，也就不知道它的构造方法，所以没有办法new一个对象，只能通过反射来进行创建
- 泛型的上下限
	- 为什么要框定上下限
		- ```
		  class A{}
		  class B extends A {}
		  
		  // 如下两个方法不会报错
		  public static void funA(A a) {
		      // ...          
		  }
		  public static void funB(B b) {
		      funA(b);
		      // ...             
		  }
		  
		  // 如下funD方法会报错
		  public static void funC(List<A> listA) {
		      // ...          
		  }
		  public static void funD(List<B> listB) {
		      funC(listB); // Unresolved compilation problem: The method doPrint(List<A>) in the type test is not applicable for the arguments (List<B>)
		      // ...             
		  }
		  ```
		- 解决方法
			- 加入了类型的上下边界机制。extends A，A是上界，能用A及A的子类。
			- 编译的时候，用A类型代替类型参数。
		- 规定
			- extends A，上界，A及A的子类能用
			- super A A是下届，A及A的父类能用
		- 小结
			- ```
			  <?> 无限制通配符
			  <? extends E> extends 关键字声明了类型的上界，表示参数化的类型可能是所指定的类型，或者是此类型的子类
			  <? super E> super 关键字声明了类型的下界，表示参数化的类型可能是指定的类型，或者是此类型的父类
			  
			  // 使用原则《Effictive Java》
			  // 为了获得最大限度的灵活性，要在表示 生产者或者消费者 的输入参数上使用通配符，使用的规则就是：生产者有上限、消费者有下限
			  1. 如果参数化类型表示一个 T 的生产者，使用 < ? extends T>;
			  2. 如果它表示一个 T 的消费者，就使用 < ? super T>；
			  3. 如果既是生产又是消费，那使用通配符就没什么意义了，因为你需要的是精确的参数类型。
			  ```
		- 要限制多个：用&符号进行链接
- 泛型数组
	- ```
	  public ArrayWithTypeToken(Class<T> type, int size) {
	      array = (T[]) Array.newInstance(type, size);
	  }
	  ```
-
- 深入理解泛型
	- 类型擦除
		- 删除<>及其保卫的部分
		- 如果无限制就替换成object，如果规定了上界就取上界，规定了下界就取object
		- 为了保证类型安全，必要的时候插入强制类型转换代码
		- 自动产生“桥接方法”以保证擦除类型后的代码仍然具有泛型的“多态性”。
	- 使用类型
		- 不指定泛型的情况下，泛型变量的类型是方法中几种类型的同一父类的最小集
		- 指定泛型的情况下，方法的几种类型必须是该泛型的实例的类型或者其子类
	- 如何理解泛型的编译期检查
		- ```
		  ArrayList<String> list1 = new ArrayList(); //第一种 情况
		  ArrayList list2 = new ArrayList<String>(); //第二种 情况
		  ```
		- 类型检查时编译时完成的，new ArrayList()在内存中开辟了一个存储空间，可以存储任何类型对象，真正涉及类型检查的是引用。
		- **类型检查就是针对引用的，谁是一个引用，用这个引用调用泛型方法，就会对这个引用调用的方法进行类型检测，而无关它真正引用的对象**。
		- 泛型中参数化类型不考虑继承关系
		- ```
		  ArrayList<String> list1 = new ArrayList<Object>(); //编译错误
		  用list1.get()值的时候，里面都是string类型的对象，但是实际上里面存的是object类型队形
		  ArrayList<Object> list2 = new ArrayList<String>(); //编译错误
		  ```
- 泛型的多态
	- 有这样的泛型类
	- ```
	  class Pair<T> {  
	  
	      private T value;  
	  
	      public T getValue() {  
	          return value;  
	      }  
	  
	      public void setValue(T value) {  
	          this.value = value;  
	      }  
	  }
	  ------
	  著作权归@pdai所有
	  原文链接：https://pdai.tech/md/java/basic/java-basic-x-generic.html
	  ```
	- 要子类继承它
		- ```
		  class DateInter extends Pair<Date> {  
		  
		      @Override  
		      public void setValue(Date value) {  
		          super.setValue(value);  
		      }  
		  
		      @Override  
		      public Date getValue() {  
		          return super.getValue();  
		      }  
		  }
		  ------
		  著作权归@pdai所有
		  原文链接：https://pdai.tech/md/java/basic/java-basic-x-generic.html
		  ```
		- 但是类型擦除之后，父类的泛型类型变成了object。但是子类的类型是Date，参数不一样了，感觉是重载了，但是如果是重载了，子类也没有继承父类的object类型参数的方法。
		- JVM采用了桥方法
			- 编译器自己生成了桥方法，桥方法的参数类型是object。内部实现是调用我们自己重写的那两方法
			- 子类中的桥方法`Object getValue()`和`Date getValue()`是同时存在的，可是如果是常规的两个方法，他们的方法签名是一样的，也就是说虚拟机根本不能分别这两个方法。如果是我们自己编写Java代码，这样的代码是无法通过编译器的检查的，但是虚拟机却是允许这样做的，因为虚拟机通过参数类型和返回类型来确定一个方法，所以编译器为了实现泛型的多态允许自己做这个看起来“不合法”的事情，然后交给虚拟器去区别。
- 基本类型不能作为泛型类型。因为object类型不能存储int值
- 泛型类型不能实例化。因为类型擦除了之后，都会变成object，如果能new，失去了意义
- 泛型数组初始化的时候数组类型不能是具体的泛型类型，只能是通配符的形式，具体类型会导致可以存入任意类型对象，在取出的时候会发生类型转换异常。
- 通过反射来进行定义
	- ```
	  v、public class ArrayWithTypeToken<T> {
	      private T[] array;
	  
	      public ArrayWithTypeToken(Class<T> type, int size) {
	          array = (T[]) Array.newInstance(type, size);
	      }
	  
	      public void put(int index, T item) {
	          array[index] = item;
	      }
	  
	      public T get(int index) {
	          return array[index];
	      }
	  
	      public T[] create() {
	          return array;
	      }
	  }
	  //...
	  
	  ArrayWithTypeToken<Integer> arrayToken = new ArrayWithTypeToken<Integer>(Integer.class, 100);
	  Integer[] array = arrayToken.create();
	  ------
	  著作权归@pdai所有
	  原文链接：https://pdai.tech/md/java/basic/java-basic-x-generic.html
	  ```
- 泛型类中的静态方法和静态变量不可以使用泛型类所声明的泛型类型参数
	- 但是泛型方法可以，因为泛型方法中使用的T是自己在方法中定义的T，而不是泛型类中的T
- 不能抛出也不能捕获泛型类的对象。泛型类扩展throwable也不合法
- catch字句里不能使用泛型变量
-
-
-
-
-
- 重载
	- 返回值不一样不行，要方法名相同，参数个数或类型或顺序不一样才行
- 问题：
	- 自动产生“桥接方法”以保证擦除类型后的代码仍然具有泛型的“多态性”。 啥意思