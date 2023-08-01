
校验 & 数据库连接池
===============
数据库连接池
> * 单例模式，保证唯一
> * list实现连接池
> * 连接池为静态大小
> * 互斥锁实现线程安全

校验  
> * HTTP请求采用POST方式
> * 登录用户名和密码校验
> * 用户注册及多线程注册安全

### 什么是连接池？

#### 什么是"池"

所谓的"池"实际上就是**一组资源的集合**，任何资源如果有需要都可以以池的形式组织，比如**线程池**

简单来说，**池是资源的容器**，本质上是对资源的复用。连接池也不例外

连接池中的资源为一组**数据库连接**，由程序动态地对池中的连接进行使用，释放。



#### 为什么要将"数据库连接"通过"池"来管理？

数据库访问的流程一般是：当系统需要访问数据库时，先系统创建数据库连接，完成数据库操作，然后系统断开数据库连接。

按照上面的流程，如果要频繁地访问数据库，那就得不断创建和断开数据库连接，这个过程很耗时且存在数据安全隐患。

所以，在程序初始化时我们就提前创建一些数据库连接，将它们用池管理起来，等用的时候再给程序，这样既能保证较快的数据库访问速度，又能确保数据安全。



### <a name="section11">连接池类</a>

上代码

主函数`main.cpp`中通过调用webserver对象的`sql_pool()`函数来创建一个连接池

```c++
server.sql_pool();
```

`void WebServer::sql_pool()`首先创建一个连接池对象，然后进行初始化操作

```c++
void WebServer::sql_pool(){
    //初始化数据库连接池
    m_connPool = connection_pool::GetInstance();
    m_connPool->init("localhost", m_user, m_passWord, m_databaseName, 3306, m_sql_num, m_close_log);

    //初始化数据库读取表
    users->initmysql_result(m_connPool);
}
```

看到这个"GetInstance()"想到什么？没错，单例模式

这个数据库连接池的设计也使用到了单例模式，前面日志处理部分的时候我们见识过单例模式了其实

```c++
connection_pool *connection_pool::GetInstance(){
	static connection_pool connPool;
	return &connPool;
}//GetInstance()被调用之后返回一个唯一的连接池实例
```

并且这里很明显使用的也是懒汉式，GetInstance()被调用才会创建或返回唯一实例

（区分懒汉还是饿汉，最好就是看该类在头文件中的定义）

#### 初始化

```c++
//构造初始化
void connection_pool::init(string url, string User, string PassWord, string DBName, int Port, int MaxConn, int close_log){
	m_url = url;
	m_Port = Port;
	m_User = User;
	m_PassWord = PassWord;
	m_DatabaseName = DBName;
	m_close_log = close_log;
    ...
```

传入的参数赋值给连接池的成员变量，包括主机地址(`m_url`)、端口号(`m_Port`)、用户名(`m_User`)、密码(`m_PassWord`)、数据库名(`m_DatabaseName`)和日志开关(`m_close_log`)。

然后使用循环创建指定数量(`MaxConn`)的数据库连接对象，并将其添加到连接池的`connList`列表中。

```c++
	...
	for (int i = 0; i < MaxConn; i++){
		MYSQL *con = NULL;
		con = mysql_init(con);

		if (con == NULL){
			LOG_ERROR("MySQL Error");
			exit(1);
		}
		con = mysql_real_connect(con, url.c_str(), User.c_str(), PassWord.c_str(), DBName.c_str(), Port, NULL, 0);

		if (con == NULL){
			LOG_ERROR("MySQL Error");
			exit(1);
		}
		connList.push_back(con);
		++m_FreeConn;//每创建一个连接对象，空闲连接数(m_FreeConn)加1
	}//信号量reserve初始化，将信号量的初始值设置为空闲连接数(m_FreeConn)，用于控制连接的获取。
	reserve = sem(m_FreeConn);

	m_MaxConn = m_FreeConn;//将最大连接数(m_MaxConn)设置为当前空闲连接数(m_FreeConn)。
}
```

这里首先使用`mysql_init()`（MySQL C API 提供的函数）初始化一个 MYSQL 结构体对象。

`con`是一个 MYSQL 结构体指针，传递给`mysql_init()`函数后，函数会初始化`con`指向的MYSQL结构体对象，使其成为一个有效的、用于表示数据库连接的对象。该对象可进行后续的数据库操作，如连接数据库、执行 SQL 查询等。

然后，我们需要使用`mysql_real_connect（）`给`con`提供用于连接数据库的信息。如果连接成功，`mysql_real_connect()`函数返回一个非空的MYSQL结构体指针，表示连接成功的连接对象。如果连接失败，返回 NULL。

在给定的代码中，连接成功后，将连接对象添加到连接池的`connList`列表(`list<MYSQL *> connList;`)中，并增加空闲连接数。

> 注意，在使用完连接对象后，需要通过`mysql_close()`函数关闭连接，并将连接对象从连接池中移除。这部分逻辑在`ReleaseConnection()`和`DestroyPool()`（[详见](#section10)）函数中实现。



#### 使用数据库

初始化完数据库连接，并将其加入连接池后，当然得用这个连接去访问数据库啦

```c++
void WebServer::sql_pool(){
    //初始化数据库连接池
    ...
    //初始化数据库读取表
    users->initmysql_result(m_connPool);
}
```

这里调用的是http_conn对象users中的initmysql_result()函数来与数据库进行交互

在该函数中，创建`connectionRAII`对象`mysqlcon`，并传递`&mysql`和`connPool`作为参数。这样，`mysqlcon`对象的构造函数会获取一个数据库连接并将其赋值给`mysql`。

```c++
map<string, string> users;
void http_conn::initmysql_result(connection_pool *connPool){
    //先从连接池中取一个连接
    MYSQL *mysql = NULL;
    connectionRAII mysqlcon(&mysql, connPool);
	...
```

`connectionRAII`是一个自定义的类，用于管理数据库连接的生命周期。它的构造函数接受两个参数：一个`MYSQL**`类型的指针`con`和一个`connection_pool*`类型的指针`connPool`。

简单来说，我们通过connectionRAII类来获取连接池中保存的"连接"资源，然后供后续使用。

> 注意，这里创建的connectionRAII类负责管理一个连接的整个使用周期，包括其取用到销毁
>
> 详见[connectionRAII类介绍](#section11)

从连接池拿到连接后，开始检索数据库

```c++
	...
	//在user表中检索username，passwd数据，浏览器端输入
    if (mysql_query(mysql, "SELECT username,passwd FROM user")){
        LOG_ERROR("SELECT error:%s\n", mysql_error(mysql));
    }
```

首先查询的是用户从浏览器输入的账户名和密码

然后查询剩余的信息并返回

从mysql中可以得到所有数据的返回值，此时选取row[0]和row[1]对应着用户名和密码，将其保存到users（http_conn对象）中相应的成员属性中。

OK，现在我们完成了以下流程：

用户在浏览器输入用户名、密码进行注册->用户名密码入库，注册完成->用户输入用户名密码登录->使用输入数据在数据库查询

实际上到这里，连接池类的工作就已经完成了

#### 总结

##### 连接池类做了什么？

字面意思，连接池类使用单例模式进行设计，创建了一个并维护一个连接池。

但是在初始化之后，维护连接池的工作更多的是由connectionRAII类来完成，该类调用连接池类提供的唯一实例，对外部提供了获取和使用连接池中连接的方法。

##### 什么是RAII？

读完代码之后可以发现，实际上真正的连接池类`connection_pool`好像并没有被"直接"使用，就连连接池的管理都是"外包"出去的

这个就是RAII机制

RAII（Resource Acquisition Is Initialization）是一种资源获取即初始化的编程技术，用于管理资源的生命周期。它是一种 C++ 的编程范式，通过在对象的构造函数中获取资源，并在对象的析构函数中释放资源，以确保资源在对象的生命周期内得到正确的管理和释放。

RAII的基本原则是：在对象的构造函数中获取资源，并在析构函数中释放资源。通过这种方式，无论是正常执行还是异常情况下的退出，都可以保证资源的正确释放，避免资源泄漏。

对应到连接池的设计中就是：

连接池是保存"连接"这种资源的一个容器，而我们在创建一个连接池对象后，不直接通过对象提供的函数来使用池中的连接。

我们定义一个类connectionRAII，

该类的**构造函数**中调用连接池类提供的GetConnection()函数来获取连接；

该类的**析构函数**中调用连接池类提供的ReleaseConnection()函数来获取连接；

简单来说，**RAII就是给某些资源类再次进行了封装，让使用者专注于资源的使用逻辑而不需要考虑资源的管理细节，降低资源泄漏的概率**

##### RAII这么好为什么线程池不用？

在 Web 服务器中，线程池用于管理并发处理客户端请求的线程。

通常情况下，**线程池中的线程需要保持活动状态，以便随时处理新的请求**。如果使用 RAII 来管理线程池中的线程，那么在**线程对象的析构函数中释放线程资源将导致线程被终止，从而无法继续处理新的请求**。

> 为了保持线程池的正常工作和线程的重用，一般不使用 RAII 来管理线程池中的线程。相反，通常会使用其他手段来管理线程的生命周期，例如使用条件变量或标志位来控制线程的启动和停止，或者使用特定的线程池管理类来管理线程的创建、启动和销毁。

这就是为什么线程池不用RAII的原因


