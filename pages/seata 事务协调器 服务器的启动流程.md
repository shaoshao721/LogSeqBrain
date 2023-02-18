tags:: seata，事务协调器

- 事务协调器 seata server
	- 步骤
		- 创建metrics解释器
		- 吃刷metrics解释器，用来进行采集，监控seata的一些运行指标，可以和外部的一些监控系统对接
		- 初始化UUID产生器，用来生成集群内不重复的唯一ID，包括全局事务ID，分支事务ID
		- 初始化sessionHolder，负责session的持久化，一个session对象是一个事务。sessionHolder支持文件，数据库，redis这三种持久化方式
		- 初始化事务协调器
		- 初始化rpc server，基于netty实现的rpc服务器。初始化netty，设置channelhandler，启动netty。把自己的ip地址，端口号注册到注册中心，客户端能够找到它