tags:: spring

- spring的底层源码启动过程
- BeanDefinition，BeanPostProcessor的概念
- Spring解析配置类等底层源码工作流程
- 依赖注入，Aware回调等底层源码工作流程
- springAOP的底层源码工作流程
-
- 在spring容器启动的时候，就会去把非懒加载的单例Bean都加载出来。如果在懒加载的单例Bean(@Lazy)的话，会在getBean的时候才会去加载