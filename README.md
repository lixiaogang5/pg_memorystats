
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
- PQconnectdb() 与数据库服务器建立新连接。 
其函数原型如下：
```c
PGconn *PQconnectdb(const char *conninfo)
``` 
该例程使用从字符串conninfo中获取的参数打开一个新的数据库连接。与下面的PQsetdbLogin（()不同，可以在不更改功能签名的情况下扩展参数集，因此对于应用程序编程，首选使用此例程或使用非阻塞类似物PQconnectStart / PQconnectPoll。传递的字符串可以为空以使用所有默认参数，也可以包含一个或多个用空格分隔的参数设置。

每个参数设置都采用key = value 的形式。（要写一个空值或包含空格的值，请用单引号将其括起来，例如，keyword = 'a value'。该值内的单引号必须写为\'。等号周围的空格是可选的。）当前公认的参数关键字是：  

- host  
要连接的主机名。如果指定了非零长度的字符串，则使用TCP / IP通信。使用此参数将导致主机名查找。请参阅hostaddr。
- hostaddr  
要连接的主机的数字IP地址。这应该采用标准的IPv4地址格式，例如192.168.40.9。如果您的计算机支持IPv6，则也可以使用这些地址。当为此参数指定非空字符串时，始终使用TCP/IP通信。  
使用hostaddr代替host可以使应用程序避免查找主机名，这在有时间限制的应用程序中可能很重要。但是，Kerberos身份验证需要主机名。因此，以下适用：  
(1) 如果在没有hostaddr的情况下指定了host，则将强制进行主机名查找。   
(2) 如果在没有host的情况下指定了hostaddr，则hostaddr的值将提供远程地址。如果使用Kerberos，则将导致反向名称查询。如果同时指定了host和hostaddr，则hostaddr的值将提供远程地址；否则，该值为0。除非使用Kerberos，否则将忽略host的值，在这种情况下，该值将用于Kerberos身份验证。请注意，如果libpq传递的主机名不是hostaddr上计算机的名称，则身份验证很可能会失败。

没有host或hostaddr， libpq将使用本地Unix域套接字进行连接。或在没有Unix域套接字的计算机上，它将尝试连接到localhost。

- port  
要在服务器主机上连接的端口号，或用于Unix域连接的套接字文件扩展名。   
- dbname  
数据库名称。默认值与用户名相同。 
- user   
用户用来与基础数据库建立连接的EnterpriseDB用户名。它的默认值与运行该应用程序的用户的操作系统所使用的值相同。
- password  
如果服务器要求密码验证，则使用密码。
- options  
将发送到服务器的跟踪/调试选项。
- tty  
该参数已启用忽略（以前，此选项指定将服务器调试输出发送到的位置）。

如下图所示：

![3  PQconnectdb()函数的公认参数关键字](https://user-images.githubusercontent.com/63132178/185537883-1455d535-230b-45f0-a5f8-64b19f2aed46.png)
除了上面列出的几个常用连接参数关键字之外，还有：sslmode、requiressl、service、krbsrvname、authtype、connect_timeout、keepalives_interval等共几十个连接选项参数，剩下的会在后面介绍PQconnectdb()函数的时候进行详细介绍。



