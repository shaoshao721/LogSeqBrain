tags:: seata，事务协调器

- TCInboundHandler接口定义了关于全局事务开始，提交，回滚。分支事务注册和上报状态，全局锁查询，全局事务状态查询与上报处理
- 全局事务开始事件处理过程
- ```
  	    protected void doGlobalBegin(GlobalBeginRequest request, GlobalBeginResponse response, RpcContext rpcContext)
              throws TransactionException {
          // core.begin方法创建一个全局事务，得到全局事务xid
          response.setXid(core.begin(rpcContext.getApplicationId(), rpcContext.getTransactionServiceGroup(),
                  request.getTransactionName(), request.getTimeout()));
          if (LOGGER.isInfoEnabled()) {
              LOGGER.info("Begin new global transaction applicationId: {},transactionServiceGroup: {}, transactionName: {},timeout:{},xid:{}",
                      rpcContext.getApplicationId(), rpcContext.getTransactionServiceGroup(), request.getTransactionName(), request.getTimeout(), response.getXid());
          }
      }
  ```
- 全局事务提交事件过程
	- 将全局事务状态置为提交中
	- 失败了会调用异常处理方法并检查事务状态
		- ```
		  private void checkTransactionStatus(AbstractGlobalEndRequest request, AbstractGlobalEndResponse response) {
		          try {
		              // 根据xid去找对应的全局事务会话，如果找到，那状态就是全局事务会话保存的全局状态。如果没找到，表示已经完成了
		              GlobalSession globalSession = SessionHolder.findGlobalSession(request.getXid(), false);
		              if (globalSession != null) {
		                  response.setGlobalStatus(globalSession.getStatus());
		              } else {
		                  response.setGlobalStatus(GlobalStatus.Finished);
		              }
		          } catch (Exception exx) {
		              LOGGER.error("check transaction status error,{}]", exx.getMessage());
		          }
		      }
		  ```
		- 已完成的情况，全局事务超时了，事务协调器的事务超时检查定时任务发现超时了，推进回滚完成，在完成后才收到全局事务提交请求
		- 网络故障，重复发送了提交请求，其实之前那次提交请求就已经完成了
		- ```
		  public GlobalStatus commit(String xid) throws TransactionException {
		          // 根据xid找到对应的全局事务会话
		          GlobalSession globalSession = SessionHolder.findGlobalSession(xid);
		          if (globalSession == null) {
		              return GlobalStatus.Finished;
		          }
		          if (globalSession.isTimeout()) {
		              LOGGER.info("TC detected timeout, xid = {}", globalSession.getXid());
		              return GlobalStatus.TimeoutRollbacking;
		          }
		          // 添加会话生命周期监听器
		          globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
		          boolean shouldCommit = SessionHolder.lockAndExecute(globalSession, () -> {
		              if (globalSession.getStatus() == GlobalStatus.Begin) {
		                  // 关闭全局事务会话，已经进入了提交环节，就不应该再注册新的分支事务了
		                  globalSession.closeAndClean();
		                  // 如果允许异步提交场景，则异步提交来提高性能
		                  if (globalSession.canBeCommittedAsync()) {
		                      globalSession.asyncCommit();
		                      MetricsPublisher.postSessionDoneEvent(globalSession, GlobalStatus.Committed, false, false);
		                      return false;
		                  } else {
		                      // 同步提交全局事务，将全局事务状态更改成提交中
		                      globalSession.changeGlobalStatus(GlobalStatus.Committing);
		                      return true;
		                  }
		              }
		              return false;
		          });
		          // 如果现在提交
		          if (shouldCommit) {
		              boolean success = doGlobalCommit(globalSession, false);
		              //提交成功，且还有剩余分支，可以异步提交，就替补提交
		              if (success && globalSession.hasBranch() && globalSession.canBeCommittedAsync()) {
		                  globalSession.asyncCommit();
		                  return GlobalStatus.Committed;
		              } else {
		                  // 返回全局事务会话中保存的状态
		                  return globalSession.getStatus();
		              }
		          } else {
		              return globalSession.getStatus() == GlobalStatus.AsyncCommitting ? GlobalStatus.Committed : globalSession.getStatus();
		          }
		      }
		  ```
-