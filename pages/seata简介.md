tags:: seata

- AT模式：非侵入式模式，自动模式，引入注解就不用再额外开发
- 包含的角色
	- TM transaction manager 事务管理器 与TC交互，开启，提交，回滚全局事务
	- RM  resource manager 资源管理器 与TC交互，负责资源的相关处理，包括分支事务注册，分支事务状态的上报
	- TC transaction coordinator 事务协调器。维护全局事务和分支事务的状态，推进事务两阶段处理。对于at模式的分支事务，tm负责事务并发控制
	- TM 和RM 通过SDK的形式作为seata的客户端与业务系统集成在一起
	- TC 作为seata的服务端独立部署
- 处理分布式事务的主要流程
	- TM开启全局事务，获得xid
	- 各个事务的参与者通过RM和资源进行交互，向TC注册分支事务
	- 去完成资源操作，上报分支事务的状态
	- TM结束全局事务，事务一阶段结束（TM向TC提交/回滚全局事务
	- TC推进事务二阶段操作，向各个RM发起二阶段提交/回滚
- seata的事务模式
	- AT
		- 自动模式，用户只需要关注自己的业务SQL语句
		- AT模式中只要增加事务注解@GlobalTransactional
	- TCC
		- 要用户自己去实现try，confirm，cancel等方法
		- 将每组tcc服务接口当做一个资源。
		- 接口的发布方，会在业务启动的时候，向TC注册TCC resource，带有一个资源ID
		- 接口的调用方，会增加一个切面，运行的时候会拦截调用。每调用try接口，会先向tc注册一个分支事务，然后执行try里面的业务逻辑，然后向tc汇报分支事务的状态。
		- 调用完成后，发布方会通知TC进行提交或滚回分布式事务，进入二阶段调用流程。TC会回调参与者，执行对应的confirm或cancel方法
		- 主要操作包括
			- 扫描tcc服务接口
			- 注册资源
			- 拦截调用
			- 注册分支事务
			- 汇报分支事务的状态
			- 回调二阶段接口
		- TCC和AT的主要区别
			- 使用上，tcc依赖用户自己实现，try，confirm，cancel方法，成本比较大。at依赖全局事务注解和代理数据源，代码不需要改动，对业务没有侵入
			- tcc模式作用范围在应用层。需要将业务模型拆分成两阶段实现，对业务逻辑的正向和反向方法。at作用在底层的数据源，通过保存操作记录的前后镜像和生成反向sql语句来进行补偿操作，对上层应用时透明的
			- tcc事务并发控制要业务自己加锁，at由seata自己加锁，加锁的粒度是全局行锁。
	- SAGA
		- 补偿协议，每一个流程都会有个对应的逆向回滚流程
		- 如果正向操作都成功了，就提交事务
		- 如果有一个错了，就开始一个个执行逆向操作，直到回到初始状态。
		- 也需要开发者自己开发的。
		- 优势
			- 一阶段提交本地数据库事务，没有锁，性能高
			- 参与者可以采用事件驱动异步执行，吞吐高
			- 易于理解和实现，
		- 缺点
			- 一阶段提交了本地数据库事务，没有预留动作，不能保证隔离，不容易进行并发控制
	- XA
		- 要在seata定义的分布式事务范围内，利用事务资源实现对xa协议的支持，用xa协议的机制管理分支事务。
		-
	-
-