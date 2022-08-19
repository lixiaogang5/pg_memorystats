
@[toc]

# 1. 概述
在开始本篇正文内容之前，强烈建议下面的这篇文章，以对`libpq`的`sock`连接管理句柄有所了解：  
[【0114】libpq连接句柄PGconn介绍](https://blog.csdn.net/lixiaogang_theanswer/article/details/126333526?spm=1001.2014.3001.5501)

另外可扩展阅读下面的这三篇文章：

[【0098】 【libpq】libpq是什么？如何使用？](https://blog.csdn.net/lixiaogang_theanswer/article/details/124942320?spm=1001.2014.3001.5501) 
[【0012】  如何使用libpq库函数集合](https://blog.csdn.net/lixiaogang_theanswer/article/details/115115348?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522166046745916781432945312%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=166046745916781432945312&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-2-115115348-null-null.nonecase&utm_term=libpq&spm=1018.2226.3001.4450)  
[【0013】   如何编译libpq.so动态库文件](https://blog.csdn.net/lixiaogang_theanswer/article/details/115307945?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522166046745916781432945312%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=166046745916781432945312&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-3-115307945-null-null.nonecase&utm_term=libpq&spm=1018.2226.3001.4450)  

# 2.  数据库连接函数
下面的例程处理与`postgres`后端服务器的连接。应用程序可以同时打开多个后端连接。(这样做的一个原因是为了访问多个数据库。)每个连接都由`PGconn`（[【0114】libpq连接句柄PGconn介绍](https://blog.csdn.net/lixiaogang_theanswer/article/details/126333526?spm=1001.2014.3001.5501)）对象表示，该对象从`PQconnectdb()`或`PQsetdbLogin()`获得。请注意，这些函数将始终返回一个非空对象指针，除非可能内存太少，甚至无法分配`PGconn`对象。在通过连接对象发送查询之前，应该调用`PQstatus()`函数来检查连接是否成功建立。

## 2.1 PQconnectdb()
