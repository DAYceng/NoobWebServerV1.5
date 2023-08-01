# NoobWebServerV1.5
这是一个web服务器（仅用于学习Linux socket编程和计算机网络），具有以下特点：

* 使用非阻塞IO实现了reactor网络模型；
* 由事件监听+事件循环组成并发框架；
* 通过线程池管理工作线程以处理客户端请求；
* 使用双向链表实现定时器，管理连接生命周期；
* 使用单例模式（懒汉式）实现一个日志处理模块，维护一个阻塞队列，使用生产者-消费者模式进行数据交互；
* 实现一个简单的在线oj，并尝试将其部署与服务器上（默认使用cpp-httplib部署）


参考：
1、游双[Linux高性能服务器编程](http://www.baidu.com/link?url=r-mQW_6e8k96qt-5rOxLjoBaET_W8t20oEWuInB6izQLf5GYkRlKyf4muPJNWTj7FgLy8qf103Whkgib_hqHXa)
2、[TinyWebServer](https://github.com/qinguoyi/TinyWebServer)
