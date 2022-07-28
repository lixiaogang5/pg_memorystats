
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

- （2） --single
执行postgres时候，如果选项参数是“--single”，则走





















