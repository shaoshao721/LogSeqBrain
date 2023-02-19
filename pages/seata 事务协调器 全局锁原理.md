tags:: seata,事务协调器，全局锁

- 专门为AT模式设计的，其他的都要用自己的方式进行并发控制
- seta全局锁，是在分支事务的隔离级别的基础上实现的全局事务的隔离性。如果数据库本地隔离级别为读已提交，那全局事务默认在读未提交隔离级别上
- 写隔离的操作顺序
	- 获取数据库锁，可以修改本地数据，但是不能提交本地事务
	- 获取全局锁，可以提交本地事务进行持久化
	- 提交本地事务后，释放数据库锁
	- 全局事务提交或回滚后释放全局锁
- LockManager中定义的接口：为分支事务添加、释放全局锁，为全局事务释放所有分支事务持有的全局锁，检查指定的key是否已经添加了全局锁，清理所有的全局锁
- 目前实现了数据库，文件和redis的锁管理器。要自定义的话，通过AbstractLockManager对它进行扩展
- 文件锁管理器添加全局锁
	- ```
	      public boolean acquireLock(BranchSession branchSession, boolean autoCommit, boolean skipCheckLock) throws TransactionException {
	          if (branchSession == null) {
	              throw new IllegalArgumentException("branchSession can't be null for memory/file locker.");
	          }
	          String lockKey = branchSession.getLockKey();
	          if (StringUtils.isNullOrEmpty(lockKey)) {
	              // 没有需要添加全局锁的数据，返回
	              return true;
	          }
	          // 获取所有行锁
	          List<RowLock> locks = collectRowLocks(branchSession);
	          if (CollectionUtils.isEmpty(locks)) {
	              // no lock
	              return true;
	          }
	          // 添加全局锁
	          return getLocker(branchSession).acquireLock(locks, autoCommit, skipCheckLock);
	      }
	  ```
	- 行锁的信息包括全局事务id，分支事务id，资源id，表名称，主键值。
	- ```
	  先lockKey根据;分隔符按照数据库表的维度区分开来。然后表名：主键值。根据每个逐渐构建一个行锁
	  调用getLocker(branchSession).acquireLock(locks)来申请全局锁
	  ```
	- LOCK_MAP定义
		- ```
		      private static final ConcurrentMap<String/* resourceId */, ConcurrentMap<String/* tableName */,
		          ConcurrentMap<Integer/* bucketId */, BucketLockMap>>>
		          LOCK_MAP = new ConcurrentHashMap<>();
		  private final ConcurrentHashMap<String/* pk */, BranchSession/* branchSession */> bucketLockMap
		              = new ConcurrentHashMap<>();
		  bucketLockMap的主键是 行锁锁代表的数据库一行记录的主键值
		  ```
		- ```
		  public boolean acquireLock(List<RowLock> rowLocks, boolean autoCommit, boolean skipCheckLock) {
		          if (CollectionUtils.isEmpty(rowLocks)) {
		              // no lock
		              return true;
		          }
		          String resourceId = branchSession.getResourceId();
		          long transactionId = branchSession.getTransactionId();
		  
		          ConcurrentMap<BucketLockMap, Set<String>> bucketHolder = branchSession.getLockHolder();
		          ConcurrentMap<String, ConcurrentMap<Integer, BucketLockMap>> dbLockMap = CollectionUtils.computeIfAbsent(
		              LOCK_MAP, resourceId, key -> new ConcurrentHashMap<>());
		          boolean failFast = false;
		          boolean canLock = true;
		          for (RowLock lock : rowLocks) {
		              String tableName = lock.getTableName();
		              String pk = lock.getPk();
		              ConcurrentMap<Integer, BucketLockMap> tableLockMap = CollectionUtils.computeIfAbsent(dbLockMap, tableName,
		                  key -> new ConcurrentHashMap<>());
		  
		              int bucketId = pk.hashCode() % BUCKET_PER_TABLE;
		              BucketLockMap bucketLockMap = CollectionUtils.computeIfAbsent(tableLockMap, bucketId,
		                  key -> new BucketLockMap());
		              BranchSession previousLockBranchSession = bucketLockMap.get().putIfAbsent(pk, branchSession);
		              if (previousLockBranchSession == null) {
		                  // No existing lock, and now locked by myself
		                  Set<String> keysInHolder = CollectionUtils.computeIfAbsent(bucketHolder, bucketLockMap,
		                      key -> ConcurrentHashMap.newKeySet());
		                  keysInHolder.add(pk);
		              } else if (previousLockBranchSession.getTransactionId() == transactionId) {
		                  // Locked by me before
		                  continue;
		              } else {
		                  LOGGER.info("Global lock on [" + tableName + ":" + pk + "] is holding by " + previousLockBranchSession.getBranchId());
		                  try {
		                      // Release all acquired locks.
		                      branchSession.unlock();
		                  } catch (TransactionException e) {
		                      throw new FrameworkException(e);
		                  }
		                  if (!autoCommit && previousLockBranchSession.getLockStatus() == LockStatus.Rollbacking) {
		                      failFast = true;
		                      break;
		                  }
		                  if (canLock) {
		                      canLock = false;
		                      if (autoCommit) {
		                          break;
		                      }
		                  }
		              }
		          }
		          if (failFast) {
		              throw new StoreException(new BranchTransactionException(LockKeyConflictFailFast));
		          }
		          return canLock;
		      }
		  ```
- 释放全局锁
	- 通过分支事务会话获取lockHolder，根据set中保存的主键从响应的桶中依次删除
	- 放锁的时候，需要保证释放的key对应的值是自己的全局事务ID，如果不是，就不操作。
		- 这时因为事务加全局锁后长时间不释放，超过时间阈值后，允许别的事务抢锁。