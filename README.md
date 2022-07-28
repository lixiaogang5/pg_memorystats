
【0107】 启动XLOG机制（1）

# 1. 概述
下图中的流程图详细地表述了PostgreSQL数据库启动的一个过程。在终端执行postgres（postmaster）启动postgres进程时，首先会执行main()函数，该函数中会完成一系列的初始化操作，比如为TopMemoryContext、ErrorContext等全局内存上下文指针变量分配8192KB大小的存储空间，之后来到一个if()...else()分支，以确定用户想用什么样的方式去启动postgres服务。


![1  StartupXLOG()函数执行时序图 (1)](https://user-images.githubusercontent.com/63132178/181451320-a3e923af-ac2d-4f1d-9e04-4af555027fb9.png)

基于当前PostgreSQL V13.2版本中，共支持4个逻辑分支，如下面代码所示：

```c
int main(int argc, char *argv[])
{
    ...... //其他操作
    if (argc > 1 && strcmp(argv[1], "--boot") == 0)
        AuxiliaryProcessMain(argc, argv);	/* does not return */
    else if (argc > 1 && strcmp(argv[1], "--describe-config") == 0)
        GucInfoMain();			/* does not return */
    else if (argc > 1 && strcmp(argv[1], "--single") == 0)
        PostgresMain(argc, argv,
             NULL,		/* no dbname */
             strdup(get_user_name_or_exit(progname)));	/* does not return */
    else
        PostmasterMain(argc, argv); /* does not return */
    abort();					/* should not get here */
}
```
- (1) --boot
如果用户执行postgres，后面的选项参数是“--boot”，那么将走AuxiliaryProcessMain(argc, argv); 函数分支，比如：``` su postgres -c '/usr/local/postgresql132/bin/postgres --boot -D /home/soft/lixiaogang5/PostgreSQL132/data'```。注意，该函数不返回。
![image](https://user-images.githubusercontent.com/63132178/181453986-e2681b86-f5b2-4a45-8ff1-ac2060a4267b.png)

- （2）--describe-config
如果用执行postgres，后面的选项参数是“--describe-config”，那么将走GucInfoMain();函数分支。该函数的主要作用是获取GUC配置参数的相关信息，而不会启动postgres服务。这是相当安全的，因为这些都是只读的活动。如下：
![image](https://user-images.githubusercontent.com/63132178/181455016-c07efb64-d6f1-44e8-a553-cfdd5e6303ee.png)

- （3） --single

执行postgres时候，如果选项参数是“--single”，则走PostgresMain()分支，该分支也称为“单用户模式”。single模式通常会在以下场景中使用，（1）initdb节点，（2）多用户模式无法工作时，可使用单用户模式连接、登录数据库，（3）修复系统故障等。

- （4）此模式就是通常默认的开启postgres服务的逻辑分支PostmasterMain（）。
![image](https://user-images.githubusercontent.com/63132178/181463082-4b593add-8f21-4dff-bc92-20401e5b2660.png)


当进入到PostmasterMain()函数数据，会创建指定的监听ip、port监听套接字，然后初始化其他的配置参数，之后postgres服务就会进入到ServerLoop()函数，这个函数内部是一个死循环，定期轮询指定ip和port监听套接字，接收客户端的网络请求连接，同时对wal write、background write、log write等后台进程的状态进行监控。此外还会依次执行BackendStartup()、BackendRun()、PostgresMain()、InitPostgres()和StartupXLOG()等函数的处理。而StartupXLOG()函数的逻辑处理功能则是本文详细介绍的重点。


# 2. 启动XLOG机制
启动XLOG机制是由StartupXLOG()函数负责完成，该函数必须在postmaster（postgres）进程或独立后端进程启动期间执行一次。StartupXLOG()函数定义于xlog.c（ src/backend/access/transam/）中，其函数原型如下：

``` c
void StartupXLOG(void)
{} 
``` 
在PostgreSQL v13.2版本中，该函数体大概有2K行。所以阅读起来是比较耗时的，因为该函数内部又调用了若干其他函数。

![9  StartupoXLOG()函数的作用](https://user-images.githubusercontent.com/63132178/181513115-d0c8b17c-a4f5-4012-a9b4-35d813cfc861.png)

因此，将StartupXLOG()函数的内部实现分为若干篇文章来进行讲解。本文主要将解该函数中以下几个功能：  
（1）检查pg_control文件中的checkPoint内容是否有效；  
（2）打印pg_control文件中PostgreSQL数据库上一次关闭时的集群状态；  
（3）验证pg_wal/archive_status目录是否存在；  
（4）清理pg_wal目录下的一些临时WAL段；  
（5）fsync()同步整个数据目录（PGDATA路径下所有目录和文件）  








