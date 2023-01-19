tags:: seata

- 对sql库里的dataSource，Connection，Statement，PreparedStatement这四个接口都进行了再次包装
- 数据源代理类 dataSourceProxy
	- 创建数据源代理。实现了resource接口，所以资源管理器可以把它看成一个资源来进行管理。继承abstractdataSourceProxy类。这个类是实现了dataSource接口的，可以拦截业务SQL语句
	- abstractDataSourceProxy定义了一个构造方法，要求传入原始的数据源，赋值给targetDataSource对象。该类其他方法直接调用对应方法
	- 构造函数实现
		- 将原始数据源保存为targetDataSource
		- 调用初始化方法init
		- 用原始数据源创建连接。datasource.getConnection()，用这个连接得到url地址，数据库类型，用户名称等信息
		- 将本数据源代理注册到资源管理器resourceManager中
	- 资源管理器
		- AT模式中，resourceManager实现类是DataSourceManager
	-