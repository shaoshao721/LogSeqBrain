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
		- clean和removeGlobalSession都是用来回滚的方法。但是clean是清除当前线程在特定全局事务中的本地回滚状态。当本地事务无法提交，seata将当前线程的回滚状态设置成true，表示需要回滚。调用了clean之后，允许线程参与其他事务。
		- removeGlobalSession 全局事务回滚失败后将它从seata的重试队列中移除
		- enable设置为false的时候且回滚重试过程超时时，涉及事务的资源将一直被锁定，直到事务成功回滚或seata的死锁检测和解决功能检测并解决到死锁
		- 回滚全局事务方法
			- ```
			  public boolean doGlobalRollback(GlobalSession globalSession, boolean retrying) throws TransactionException {
			          boolean success = true;
			          // start rollback event
			          MetricsPublisher.postSessionDoingEvent(globalSession, retrying);
			  
			          if (globalSession.isSaga()) {
			              success = getCore(BranchType.SAGA).doGlobalRollback(globalSession, retrying);
			          } else {
			              // 按照分支事务反序遍历，对该全局事务的所有分支事务进行处理
			              Boolean result = SessionHelper.forEach(globalSession.getReverseSortedBranches(), branchSession -> {
			                  BranchStatus currentBranchStatus = branchSession.getStatus();
			                  // 如果分支事务一阶段就失败了，本地事务还没有成功提交呢，那就直接删除分支事务
			                  if (currentBranchStatus == BranchStatus.PhaseOne_Failed) {
			                      SessionHelper.removeBranch(globalSession, branchSession, !retrying);
			                      return CONTINUE;
			                  }
			                  try {
			                      // 分支事务二阶段回滚
			                      BranchStatus branchStatus = branchRollback(globalSession, branchSession);
			                      if (isXaerNotaTimeout(globalSession, branchStatus)) {
			                          LOGGER.info("Rollback branch XAER_NOTA retry timeout, xid = {} branchId = {}", globalSession.getXid(), branchSession.getBranchId());
			                          branchStatus = BranchStatus.PhaseTwo_Rollbacked;
			                      }
			                      switch (branchStatus) {
			                          // 如果分支事务回滚成功，删除该分支事务
			                          case PhaseTwo_Rollbacked:
			                              SessionHelper.removeBranch(globalSession, branchSession, !retrying);
			                              LOGGER.info("Rollback branch transaction successfully, xid = {} branchId = {}", globalSession.getXid(), branchSession.getBranchId());
			                              return CONTINUE;
			                              // 如果回滚失败且标识为不可重试，那就不充实。需要人工干预
			                          case PhaseTwo_RollbackFailed_Unretryable:
			                              SessionHelper.endRollbackFailed(globalSession, retrying);
			                              LOGGER.error("Rollback branch transaction fail and stop retry, xid = {} branchId = {}", globalSession.getXid(), branchSession.getBranchId());
			                              return false;
			                              // 回滚失败，重试
			                          default:
			                              LOGGER.error("Rollback branch transaction fail and will retry, xid = {} branchId = {}", globalSession.getXid(), branchSession.getBranchId());
			                              if (!retrying) {
			                                  globalSession.queueToRetryRollback();
			                              }
			                              return false;
			                      }
			                  } catch (Exception ex) {
			                      StackTraceLogger.error(LOGGER, ex,
			                          "Rollback branch transaction exception, xid = {} branchId = {} exception = {}",
			                          new String[] {globalSession.getXid(), String.valueOf(branchSession.getBranchId()), ex.getMessage()});
			                      if (!retrying) {
			                          globalSession.queueToRetryRollback();
			                      }
			                      throw new TransactionException(ex);
			                  }
			              });
			              // Return if the result is not null
			              if (result != null) {
			                  return result;
			              }
			          }
			  
			          // In db mode, lock and branch data residual problems may occur.
			          // Therefore, execution needs to be delayed here and cannot be executed synchronously.
			          if (success) {
			              SessionHelper.endRollbacked(globalSession, retrying);
			              LOGGER.info("Rollback global transaction successfully, xid = {}.", globalSession.getXid());
			          }
			          return success;
			      }
			  ```
- 重试提交事务。全局事务状态为提交的事务，不断尝试推进他们的分支事务二阶段提交，失败就重试直到成功为止
- 异步提交事务。
- 事务超时检查，对所有全局事务进行超时检查，发现事务处于开始状态且已经超时，将全局事务状态修改为超时回滚状态，后续由重试回滚会话管理器对这个事务进行回滚
- 批量删除资源侧事务日志。通知所有rm删除7天前的事务日志。正常的话，资源侧没有多余的事务日志，但是如果存在异常关机，则会存在rm没来得及删除二阶段提交状态的事务日志的情况。防止累及过多无用事务日志