tags:: seata，事务协调器

- AT模式的branchCommit方法。
- ```
      public BranchStatus branchCommit(GlobalSession globalSession, BranchSession branchSession) throws TransactionException {
          try {
              // 设置事务id，分支事务id，资源id，应用数据，分支事务类型，发送请求到RM
              BranchCommitRequest request = new BranchCommitRequest();
              request.setXid(branchSession.getXid());
              request.setBranchId(branchSession.getBranchId());
              request.setResourceId(branchSession.getResourceId());
              request.setApplicationData(branchSession.getApplicationData());
              request.setBranchType(branchSession.getBranchType());
              return branchCommitSend(request, globalSession, branchSession);
          } catch (IOException | TimeoutException e) {
              throw new BranchTransactionException(FailedToSendBranchCommitRequest,
                      String.format("Send branch commit failed, xid = %s branchId = %s", branchSession.getXid(),
                              branchSession.getBranchId()), e);
          }
      }
  
      protected BranchStatus branchCommitSend(BranchCommitRequest request, GlobalSession globalSession,
                                              BranchSession branchSession) throws IOException, TimeoutException {
          // rpc同步发送分支事务二阶段提交请求
          BranchCommitResponse response = (BranchCommitResponse) remotingServer.sendSyncRequest(
                  branchSession.getResourceId(), branchSession.getClientId(), request);
          return response.getBranchStatus();
      }
  ```
- 在AT中，RM通过业务数据库的本地事务能力确保了幂等性。TCC模式中的话，要在自己的confirm和cancel方法中保证幂等性