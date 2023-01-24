tags:: seata，at模式

- ![image.png](../assets/image_1674450564445_0.png)
- 二阶段
	- 当全局事务为提交状态的时候，会释放各个分支事务的全局锁，推进二阶段提交
	- 资源管理器，当收到提交指令时，删除保存的事务日志，会立即返回，去异步线程批量删除
		- 分支事务提交（branchCommit）异步线程进行分支事务二阶段提交，将Phase2Context上下文放到异步执行队列中。
		- 定时任务区拉取，按照资源ID分组。
			- 根据资源ID取得数据源代理
			- 获取一个普通数据库连接！不是数据库连接代理
			- 通过 UndoLogManagerFactory.getUndoLogManager(dataSourceProxy.getDbType())获取undolog管理器
			- 把二阶段上下文列表拆分成多个小列表，防止列表过大，造成拼接处的SQL语句过长。
			- 对每个小列表进行处理，调用deleteUndoLog方法删除一批分支事务的日志
			-