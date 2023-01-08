tags:: seata

- AT模式：非侵入式模式，自动模式，引入注解就不用再额外开发
- 包含的角色
	- TM transaction manager 事务管理器 与TC交互，开启，提交，回滚全局事务
	- RM  resource manager 资源管理器 与TC交互，负责资源的相关处理，包括分支事务注册，分支事务状态的上报
	- TC transaction coordinator 事务协调器。维护全局事务和分支事务的状态，推进事务两阶段处理。对于at模式的分支事务，tm负责事务并发控制
	- TM 和RM 通过SDK的形式作为seata的客户端与业务系统集成在一起
	- TC 作为seata的服务端独立部署
-