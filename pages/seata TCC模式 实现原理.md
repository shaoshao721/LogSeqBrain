tags:: seata, TCC模式

- 注解
	- @TwoPhaseBusinessAction通过在prepare接口上标注这个注解，表示这是个TCC服务接口
	-
	- ```
	  TCC服务的名称
	  	String name();
	  confirm方法名
	      String commitMethod() default "commit";
	  cancel方法名
	      String rollbackMethod() default "rollback";
	  ```
- TCC模式的资源注册
	- 因为seata在TCC模式下，将每个服务接口作为一个资源。所以在业务启动的时候，框架会自动扫描TCC服务接口的调用方和发布方。
	- 对于发布方，seata框架向TC注册TCC Resource
	- 全局事务扫描器
		- GlobalTransactionScanner 类来扫描全局事务的定义，继承了AbstractAutoProxyCreator抽象类，重写了wrapIfNecessary()方法。这个方法用来在spring启动的时候生成代理。
		- ```
		  protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		          // 没有通过检查，则直接返回bean
		          if (!doCheckers(bean, beanName)) {
		              return bean;
		          }
		  
		          try {
		              synchronized (PROXYED_SET) {
		                  // 如果该bean已经被代理了，则直接返回bean
		                  if (PROXYED_SET.contains(beanName)) {
		                      return bean;
		                  }
		                  interceptor = null;
		                  //检查是否是TCC自动代理
		                  if (TCCBeanParserUtils.isTccAutoProxy(bean, beanName, applicationContext)) {
		                      // 如果启用了tcc Fence，则进行初始化
		                      TCCBeanParserUtils.initTccFenceCleanTask(TCCBeanParserUtils.getRemotingDesc(beanName), applicationContext);
		                      // TCC拦截器
		                      interceptor = new TccActionInterceptor(TCCBeanParserUtils.getRemotingDesc(beanName));
		                      ConfigurationCache.addConfigListener(ConfigurationKeys.DISABLE_GLOBAL_TRANSACTION,
		                              (ConfigurationChangeListener)interceptor);
		                  } else {
		                      Class<?> serviceInterface = SpringProxyUtils.findTargetClass(bean);
		                      Class<?>[] interfacesIfJdk = SpringProxyUtils.findInterfaces(bean);
		                      // 如果在类和接口中都不存在全局事务注解@GlobalTransactional，则直接返回bean
		                      if (!existsAnnotation(new Class[]{serviceInterface})
		                          && !existsAnnotation(interfacesIfJdk)) {
		                          return bean;
		                      }
		                      // 全局事务拦截器
		                      if (globalTransactionalInterceptor == null) {
		                          globalTransactionalInterceptor = new GlobalTransactionalInterceptor(failureHandlerHook);
		                          ConfigurationCache.addConfigListener(
		                                  ConfigurationKeys.DISABLE_GLOBAL_TRANSACTION,
		                                  (ConfigurationChangeListener)globalTransactionalInterceptor);
		                      }
		                      interceptor = globalTransactionalInterceptor;
		                  }
		  
		                  LOGGER.info("Bean[{}] with name [{}] would use interceptor [{}]", bean.getClass().getName(), beanName, interceptor.getClass().getName());
		                  if (!AopUtils.isAopProxy(bean)) {
		                      bean = super.wrapIfNecessary(bean, beanName, cacheKey);
		                  } else {
		                      AdvisedSupport advised = SpringProxyUtils.getAdvisedSupport(bean);
		                      Advisor[] advisor = buildAdvisors(beanName, getAdvicesAndAdvisorsForBean(null, null, null));
		                      int pos;
		                      for (Advisor avr : advisor) {
		                          // Find the position based on the advisor's order, and add to advisors by pos
		                          pos = findAddSeataAdvisorPosition(advised, avr);
		                          advised.addAdvisor(pos, avr);
		                      }
		                  }
		                  PROXYED_SET.add(beanName);
		                  return bean;
		              }
		          } catch (Exception exx) {
		              throw new RuntimeException(exx);
		          }
		      }
		  ```
		- 判断这个bean是否是TCC自动代理，如果是的话，拦截器初始化为TCC拦截器。如果bean声明了全局事务注解，则将拦截器初始为全局事务拦截器。
		- 如何判断是否是TCC自动代理
			- ```
			      public static boolean isTccAutoProxy(Object bean, String beanName, ApplicationContext applicationContext) {
			          // 判断是否为远程bean
			          boolean isRemotingBean = parserRemotingServiceInfo(bean, beanName);
			          //获取远程bean的描述信息
			          RemotingDesc remotingDesc = DefaultRemotingParser.get().getRemotingBeanDesc(beanName);
			          //如果是远程bean的话
			          if (isRemotingBean) {
			              // 如果描述信息不为空，则说明是远程bean，且为消费者或服务者，如果protocol是IN_JVM说明是LocalTcc
			              if (remotingDesc != null && remotingDesc.getProtocol() == Protocols.IN_JVM) {
			                  //LocalTCC
			                  return isTccProxyTargetBean(remotingDesc);
			              } else {
			                  // sofa:reference / dubbo:reference, factory bean
			                  return false;
			              }
			          } else {
			              if (remotingDesc == null) {
			                  //check FactoryBean
			                  if (isRemotingFactoryBean(bean, beanName, applicationContext)) {
			                      remotingDesc = DefaultRemotingParser.get().getRemotingBeanDesc(beanName);
			                      return isTccProxyTargetBean(remotingDesc);
			                  } else {
			                      return false;
			                  }
			              } else {
			                  return isTccProxyTargetBean(remotingDesc);
			              }
			          }
			      }
			  ```
		- 调用parserRemotingServiceInfo方法来解释远程服务信息，判断是否为远程bean
			- ```
			      protected static boolean parserRemotingServiceInfo(Object bean, String beanName) {
			          // 得到RemotingParser，如DubboRemotingParser,SofaRpcRemotingParser,LocalTccRemotingParser,HSFRemotingParser
			          // 遍历所有的allRemotingParsers去查找是不是这些parser
			          RemotingParser remotingParser = DefaultRemotingParser.get().isRemoting(bean, beanName);
			          if (remotingParser != null) {
			              // 调用这个方法去确定bean是否是服务的提供者或者消费者。如果是，则是远程bean，如果不是，就不是远程bean
			              return DefaultRemotingParser.get().parserRemotingServiceInfo(bean, beanName, remotingParser) != null;
			          }
			          return false;
			      }
			  ```
		- parserRemotingServiceInfo这个方法来判断是否为服务提供者或消费者。
			- ```
			  public RemotingDesc parserRemotingServiceInfo(Object bean, String beanName, RemotingParser remotingParser) {
			          // 获取服务的描述信息
			          RemotingDesc remotingBeanDesc = remotingParser.getServiceDesc(bean, beanName);
			          // 如果描述信息为空，则返回null
			          if (remotingBeanDesc == null) {
			              return null;
			          }
			          // 建立bean名字和服务描述信息的映射（最上层会用到
			          remotingServiceMap.put(beanName, remotingBeanDesc);
			  
			          Class<?> serviceClass = remotingBeanDesc.getServiceClass();
			          Method[] methods = serviceClass.getMethods();
			          // 如果bean为服务，服务提供者
			          if (remotingParser.isService(bean, beanName)) {
			              try {
			                  //如果为服务提供者，则获取到目标bean
			                  Object targetBean = remotingBeanDesc.getTargetBean();
			                  // 遍历所有的方法
			                  for (Method m : methods) {
			                      // 尝试获取目标bean
			                      TwoPhaseBusinessAction twoPhaseBusinessAction = m.getAnnotation(TwoPhaseBusinessAction.class);
			                      // 不为空，则说明声明了TCC注解，就要去注册TCC资源
			                      if (twoPhaseBusinessAction != null) {
			                          TCCResource tccResource = new TCCResource();
			                          tccResource.setActionName(twoPhaseBusinessAction.name());
			                          tccResource.setTargetBean(targetBean);
			                          tccResource.setPrepareMethod(m);
			                          tccResource.setCommitMethodName(twoPhaseBusinessAction.commitMethod());
			                          // 反射获取commit方法
			                          tccResource.setCommitMethod(serviceClass.getMethod(twoPhaseBusinessAction.commitMethod(),
			                                  twoPhaseBusinessAction.commitArgsClasses()));
			                          tccResource.setRollbackMethodName(twoPhaseBusinessAction.rollbackMethod());
			                          tccResource.setRollbackMethod(serviceClass.getMethod(twoPhaseBusinessAction.rollbackMethod(),
			                                  twoPhaseBusinessAction.rollbackArgsClasses()));
			                          // set argsClasses
			                          tccResource.setCommitArgsClasses(twoPhaseBusinessAction.commitArgsClasses());
			                          tccResource.setRollbackArgsClasses(twoPhaseBusinessAction.rollbackArgsClasses());
			                          // set phase two method's keys
			                          tccResource.setPhaseTwoCommitKeys(this.getTwoPhaseArgs(tccResource.getCommitMethod(),
			                                  twoPhaseBusinessAction.commitArgsClasses()));
			                          tccResource.setPhaseTwoRollbackKeys(this.getTwoPhaseArgs(tccResource.getRollbackMethod(),
			                                  twoPhaseBusinessAction.rollbackArgsClasses()));
			                          //注册TCC资源
			                          DefaultResourceManager.get().registerResource(tccResource);
			                      }
			                  }
			              } catch (Throwable t) {
			                  throw new FrameworkException(t, "parser remoting service error");
			              }
			          }
			          // 如果bean是服务消费者
			          if (remotingParser.isReference(bean, beanName)) {
			              //标记bean为引用
			              remotingBeanDesc.setReference(true);
			          }
			          return remotingBeanDesc;
			      }
			  ```
			- 如果是服务提供者，也声明了tcc的注解，则构建TCC资源对象，并且调用ResourceManager的registerResource方法进行TCC的资源注册
			- 资源注册，先做本地缓存，然后通过资源管理器的RPC客户端将当前资源的资源组ID和资源ID发给资源协调器，注册TCC资源
	- TCC模式的事务发起
		- TCC模式的业务调用方和AT模式的业务调用方一样，要用@GlobalTransactional来声明全局事务
		- 业务方法执行的时候，会被全局事务拦截器GlobalTransactionalInterceptor类拦截并开启一个全局事务
			- ```
			  // 2. If the tx role is 'GlobalTransactionRole.Launcher', send the request of beginTransaction to TC,
			                  //    else do nothing. Of course, the hooks will still be triggered.
			                  beginTransaction(txInfo, tx);
			  
			                  Object rs;
			                  try {
			                      // Do Your Business
			                      rs = business.execute();
			                  } catch (Throwable ex) {
			                      // 3. The needed business exception to rollback.
			                      completeTransactionAfterThrowing(txInfo, tx, ex);
			                      throw ex;
			                  }
			  
			                  // 4. everything is fine, commit.
			                  commitTransaction(tx, txInfo);
			  
			                  return rs;
			  ```
			- 开启全局事务
			- 执行业务方法
				- 如果异常了，则回滚
				- 没有异常，则提交全局事务
		- TCC拦截器
			- ```
			  public Object invoke(final MethodInvocation invocation) throws Throwable {
			          if (!RootContext.inGlobalTransaction() || disable || RootContext.inSagaBranch()) {
			              //如果不在全局事务找那个，或者关闭了全局事务服务，或者是sage分支，则执行原始方法并返回
			              return invocation.proceed();
			          }
			          Method method = getActionInterfaceMethod(invocation);
			          // 得到TCC注解
			          TwoPhaseBusinessAction businessAction = method.getAnnotation(TwoPhaseBusinessAction.class);
			          //try method
			          if (businessAction != null) {
			              //save the xid
			              String xid = RootContext.getXID();
			              //save the previous branchType
			              BranchType previousBranchType = RootContext.getBranchType();
			              //if not TCC, bind TCC branchType
			              if (BranchType.TCC != previousBranchType) {
			                  RootContext.bindBranchType(BranchType.TCC);
			              }
			              try {
			                  //执行TCC事务调用，并进行返回
			                  return actionInterceptorHandler.proceed(method, invocation.getArguments(), xid, businessAction,
			                          invocation::proceed);
			              } finally {
			                  //if not TCC, unbind branchType
			                  if (BranchType.TCC != previousBranchType) {
			                      RootContext.unbindBranchType();
			                  }
			                  //MDC remove branchId
			                  MDC.remove(RootContext.MDC_KEY_BRANCH_ID);
			              }
			          }
			  
			          //not TCC try method
			          return invocation.proceed();
			      }
			  ```
			- 判断是否在全局事务中，如果不在，执行原始方法。在，调用actionInterceptorHandler.proceed处理tcc事务调用
		- 拦截处理过程
			- ```
			      public Object proceed(Method method, Object[] arguments, String xid, TwoPhaseBusinessAction businessAction,
			                                         Callback<Object> targetCallback) throws Throwable {
			          //Get action context from arguments, or create a new one and then reset to arguments
			          BusinessActionContext actionContext = getOrCreateActionContextAndResetToArguments(method.getParameterTypes(), arguments);
			  
			          //设置全局事务id
			          actionContext.setXid(xid);
			          //获取并设置TCC服务接口的名称
			          String actionName = businessAction.name();
			          actionContext.setActionName(actionName);
			          //Set the delay report
			          actionContext.setDelayReport(businessAction.isDelayReport());
			  
			          //创建分支事务
			          String branchId = doTccActionLogStore(method, arguments, businessAction, actionContext);
			          actionContext.setBranchId(branchId);
			          //MDC put branchId
			          MDC.put(RootContext.MDC_KEY_BRANCH_ID, branchId);
			  
			          // save the previous action context
			          BusinessActionContext previousActionContext = BusinessActionContextUtil.getContext();
			          try {
			              //share actionContext implicitly
			              BusinessActionContextUtil.setContext(actionContext);
			  
			              if (businessAction.useTCCFence()) {
			                  try {
			                      // Use TCC Fence, and return the business result
			                      return TCCFenceHandler.prepareFence(xid, Long.valueOf(branchId), actionName, targetCallback);
			                  } catch (SkipCallbackWrapperException | UndeclaredThrowableException e) {
			                      Throwable originException = e.getCause();
			                      if (originException instanceof FrameworkException) {
			                          LOGGER.error("[{}] prepare TCC fence error: {}", xid, originException.getMessage());
			                      }
			                      throw originException;
			                  }
			              } else {
			                  //执行方法
			                  return targetCallback.execute();
			              }
			          } finally {
			              try {
			                  //to report business action context finally if the actionContext.getUpdated() is true
			                  BusinessActionContextUtil.reportContext(actionContext);
			              } finally {
			                  if (previousActionContext != null) {
			                      // recovery the previous action context
			                      BusinessActionContextUtil.setContext(previousActionContext);
			                  } else {
			                      // clear the action context
			                      BusinessActionContextUtil.clear();
			                  }
			              }
			          }
			      }
			  ```
			- 初始化TCC事务上下文，创建分支事务，得到分支事务ID，设置运行参数，执行try方法
		- 创建分支事务
			- ```
			      protected String doTccActionLogStore(Method method, Object[] arguments, TwoPhaseBusinessAction businessAction,
			                                           BusinessActionContext actionContext) {
			          String actionName = actionContext.getActionName();
			          String xid = actionContext.getXid();
			  
			          //获取TCC请求上下文
			          Map<String, Object> context = fetchActionRequestContext(method, arguments);
			          context.put(Constants.ACTION_START_TIME, System.currentTimeMillis());
			  
			          //初始化业务上下文
			          initBusinessContext(context, method, businessAction);
			          //初始化框架上下文
			          initFrameworkContext(context);
			  
			          Map<String, Object> originContext = actionContext.getActionContext();
			          if (CollectionUtils.isNotEmpty(originContext)) {
			              //Merge context and origin context if it exists.
			              //@since: above 1.4.2
			              originContext.putAll(context);
			              context = originContext;
			          } else {
			              actionContext.setActionContext(context);
			          }
			  
			          //endregion
			  
			          //// 设置应用上下文
			          Map<String, Object> applicationContext = Collections.singletonMap(Constants.TCC_ACTION_CONTEXT, context);
			          String applicationContextStr = JSON.toJSONString(applicationContext);
			          try {
			              //通过RM注册分支事务
			              Long branchId = DefaultResourceManager.get().branchRegister(BranchType.TCC, actionName, null, xid,
			                      applicationContextStr, null);
			              return String.valueOf(branchId);
			          } catch (Throwable t) {
			              String msg = String.format("TCC branch Register error, xid: %s", xid);
			              LOGGER.error(msg, t);
			              throw new FrameworkException(t, msg);
			          }
			      }
			  ```
			- 设置上下文，调用资源管理器注册分支事务，得到分支事务ID
		- 二阶段处理
			- io.seata.rm.tcc.TCCResourceManager#branchCommit
			- 所有分支事务一阶段工作完成后，如果事务发起方没有捕获到异常，会发起这个全局事务进行提交，TC就会将全局事务中的所有分支事务发起二阶段提交处理。
			- 先通过资源ID找到本地缓存的TCCResource，然后通过resource得到confirm方法，获取分支事务相应的业务上下文对象，执行confirm方法，完成提交
			- cancel方法类似
-