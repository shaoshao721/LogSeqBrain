tags:: seata

- 基本原理
	- 通过seata的数据源代理datasourceproxy类对数据库进行操作
	- 业务通过jdbc接口访问数据库资源的时候，数据源代理会拦截所有请求。除了执行原始请求外，还会做一些和分布式事务相关的工作，产生前，后镜像，加锁数据，保存事务日志
	- 每个参加at模式事务的数据库都是一个资源。本地事务提交前，会向tc注册一个分支事务。对应会插入一行事务日志，注册分支事务成功后，提交本地事务。提交之后，向tc汇报分支事务状态。
	- 全都执行完之后，会根据事务管理器有没有收到异常来决定提交还是回滚全局事务。
	- 事务协调器会根据分支事务id，从事务日志表找到事务日志。
- 工作流程示例
	- 场景：买了东西，要将余额减掉，同时需要去增加积分操作。
	- 分析，有两个seata全局事务。一个是余额服务，一个是积分服务。同时是一个事务嵌套，余额服务传递事务上下文到积分服务。积分服务自己声明的全局事务是不生效的。
	- 流程
		- 余额服务tm向tc申请开启一个全局事务，返回全局事务id  xid
		- 开启本地事务，生成undo log，执行业务sql语句，生成redo log，将undo log和redo log保存下来，生成全局锁数据
		- rm向tc注册分支事务
		- 提交本地事务
		- rm向tc上报事务状态
		- 远程调用，把xid传给积分服务
		- 积分服务开启本地事务，生成undo log，执行业务sql语句，生成redo log，保存。生成全局锁数据
		- 积分服务rm向tc注册服务
		- 提交本地事务
		- 积分服务rm向tc上报事务状态。
		- 积分服务返回远程调佣成功给余额服务
		- 余额服务的tm向tc申请全局事务的提交、回滚
		- 二阶段处理：
			- 提交，则tc通知各个rm异步清理本地事务日志
			- 回滚，tc通知各rm回滚数据
	- 事务日志表
		- datasourceproxy拦截业务sql语句后，生成前后镜像事务日志，把事务日志保存在事务日志表（undo_log）
		- at模式中，每个参与事务的业务数据库都会创建一张事务日志表
		- 表里的字段
			- branch_id分支事务id
			- xid 全局事务id
			- context
			- rollback_info 记录了回滚的数据信息。也会冗余两种事务id
			- log_status
			- log_created
			- log_modified
			- ext
		- afterImage中的数据。
			- tableRecords：一个表的信息
			- row：一行的信息
			- field：行中各个字段的信息
			- tableRecords：
				- ![image.png](../assets/image_1673690329096_0.png)
				- tableMeta 表元数据
					- ![image.png](../assets/image_1673690383270_0.png)
					- 包括表名称，列元数据映射，索引元数据映射
					- 列元数据：列名，列数据类型，列是否允许为空，列是否为自增字段。创建表的时候会指定这些属性
					- 索引元数据：索引名称，索引类型，索引包含的列。
				- 表元数据的获取
					- 先从缓存里拿，缓存里没有，就从数据库执行查询并生成表元数据
					- 从缓存中获取
					- ![image.png](../assets/image_1673690896925_0.png)
					- ![image.png](../assets/image_1673690966682_0.png)
					- ![image.png](../assets/image_1673691093931_0.png)
					- 在datasourceproxy类中，有个定时任务，定时调用refresh方法，将所有的表元数据进行定时put。
					- 查询都是查询fetchSchema方法
						- ![image.png](../assets/image_1673691506592_0.png)
						- 利用数据库元数据得到表的列描述信息和索引描述信息，然后columnMeta和IndexMeta的各个属性赋值
			- Row
				- 一行记录，包含多个字段
			- Field
				- 字段名称，字段是否为主键，字段类型（Types枚举定义），字段的值
		- 回滚数据的时候，看后镜像中的数据是否和当前数据一致。如果一致，就可以安全的进行sql语句的回滚
- 事务日志管理器
	- ```
	  public interface UndoLogManager {
	  
	      // 保存事务日志
	      void flushUndoLogs(ConnectionProxy cp) throws SQLException;
	  
	      // 二阶段回滚处理
	      void undo(DataSourceProxy dataSourceProxy, String xid, long branchId) throws TransactionException;
	  
	      // 二阶段回滚处理的删除事务日志
	      void deleteUndoLog(String xid, long branchId, Connection conn) throws SQLException;
	  
	      // 二阶段提交处理的批量删除事务日志
	      void batchDeleteUndoLog(Set<String> xids, Set<Long> branchIds, Connection conn) throws SQLException;
	  
	      // 根据创建时间删除事务日志
	      int deleteUndoLogByLogCreated(Date logCreated, int limitRows, Connection conn) throws SQLException;
	  
	       // 资源存在undoLog表吗？如果没用AT模式的话，可能没有。如果用了AT模式的话每个库会有一张
	      boolean hasUndoLogTable(Connection conn);
	  }
	  ```
-
-