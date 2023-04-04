tags:: spring

- spring-framework
	- ```
	  ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
	  UserService userService = (UserService) context.getBean("userService");
	  userService.test();
	  ```
	- 通过xml去定义一些扫描路径等的信息
	- 后续
		- ```
		  AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
		  
		  UserService userService = (UserService) context.getBean("userService");
		  userService.test();
		  ```
		- 根据类的定义，注解的方式来定义扫描的包路径等信息
- spring帮我们创建出来的bean和我们自己new出来的区别
	- 如果UserService中Autowired一个别的component 叫OrderService，那UserService被创建出来的时候，orderService也有值，也被创建出来了。但是如果是自己new的话，那OrderService不会被创建出来
- 创建bean的流程
	- 1. 要创建这个类的对象，一定是通过这个类的无参构造方法，生成了对象，这个时候orderService是没有值的
	  2. 通过依赖注入，给那些在这个类里声明了Autowired的属性赋值。遍历这个类的所有属性，判定这个属性上面有没有autowired的注解
	  3.