tags:: 事务协调器，seata

- DefaultCoordinator是服务端的核心组件。
- 负责各种事务消息处理，如：全局事务开始，全局事务提交，全局事务回滚，分支事务注册，分支事务状态上报。通过rpc server和远程的tm，rm通信
- 定义了5个线程池执行器
	- 重试回滚事务
	- 重试提交事务
	- 异步提交事务
	- 事务超时检查
	- 批量删除资源侧事务日志
	- ```
	      public void init() {
	          retryRollbacking.scheduleAtFixedRate(
	              () -> SessionHolder.distributedLockAndExecute(RETRY_ROLLBACKING, this::handleRetryRollbacking), 0,
	              ROLLBACKING_RETRY_PERIOD, TimeUnit.MILLISECONDS);
	  
	          retryCommitting.scheduleAtFixedRate(
	              () -> SessionHolder.distributedLockAndExecute(RETRY_COMMITTING, this::handleRetryCommitting), 0,
	              COMMITTING_RETRY_PERIOD, TimeUnit.MILLISECONDS);
	  
	          asyncCommitting.scheduleAtFixedRate(
	              () -> SessionHolder.distributedLockAndExecute(ASYNC_COMMITTING, this::handleAsyncCommitting), 0,
	              ASYNC_COMMITTING_RETRY_PERIOD, TimeUnit.MILLISECONDS);
	  
	          timeoutCheck.scheduleAtFixedRate(
	              () -> SessionHolder.distributedLockAndExecute(TX_TIMEOUT_CHECK, this::timeoutCheck), 0,
	              TIMEOUT_RETRY_PERIOD, TimeUnit.MILLISECONDS);
	  
	          undoLogDelete.scheduleAtFixedRate(
	              () -> SessionHolder.distributedLockAndExecute(UNDOLOG_DELETE, this::undoLogDelete),
	              UNDO_LOG_DELAY_DELETE_PERIOD, UNDO_LOG_DELETE_PERIOD, TimeUnit.MILLISECONDS);
	      }
	  ```
	- 初始化：先去获取锁，获取到了之后，调用定时任务的处理类。前4个定时任务默认1s执行一次。最后一个一天执行一次。
	- 重试回滚事务
		- ```
		  protected void handleRetryRollbacking() {
		          SessionCondition sessionCondition = new SessionCondition(rollbackingStatuses);
		          sessionCondition.setLazyLoadBranch(true);
		          // 从重试回滚的回话管理器中获取所有的全局事务会话
		          Collection<GlobalSession> rollbackingSessions =
		              SessionHolder.getRetryRollbackingSessionManager().findGlobalSessions(sessionCondition);
		          // 没有回滚中的全局事务，返回
		          if (CollectionUtils.isEmpty(rollbackingSessions)) {
		              return;
		          }
		          long now = System.currentTimeMillis();
		          // 遍历所有的需要回滚的全局事务
		          SessionHelper.forEach(rollbackingSessions, rollbackingSession -> {
		              try {
		                  // 如果是正在回滚中，且事务的存在时间没有过长，就下一个。2分钟+10s
		                  if (rollbackingSession.getStatus() == GlobalStatus.Rollbacking
		                      && !rollbackingSession.isDeadSession()) {
		                      return;
		                  }
		                  // 检查是否超时重试
		                  if (isRetryTimeout(now, MAX_ROLLBACK_RETRY_TIMEOUT.toMillis(), rollbackingSession.getBeginTime())) {
		                      // 如果多次回滚一直没成功，检查配置是否为回滚失败就解锁
		                      if (ROLLBACK_RETRY_TIMEOUT_UNLOCK_ENABLE) {
		                          // 默认是不允许解锁，设置成解锁可以避免异常情况下长时间持有锁阻塞别的事务
		                          rollbackingSession.clean();
		                      }
		                      // 如果多次回滚都没成功，从事务管理器里清除掉，不处理了
		                      SessionHolder.getRetryRollbackingSessionManager().removeGlobalSession(rollbackingSession);
		                      LOGGER.error("Global transaction rollback retry timeout and has removed [{}]", rollbackingSession.getXid());
		                      SessionHelper.endRollbackFailed(rollbackingSession, true, true);
		                      return;
		                  }
		                  // 添加回话生命周期w
		                  rollbackingSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
		                  // 回滚全局事务
		                  core.doGlobalRollback(rollbackingSession, true);
		              } catch (TransactionException ex) {
		                  LOGGER.error("Failed to retry rollbacking [{}] {} {}", rollbackingSession.getXid(), ex.getCode(), ex.getMessage());
		              }
		          });
		      }
		  ```
		- 回滚全局事务方法
			- ```
			  ```