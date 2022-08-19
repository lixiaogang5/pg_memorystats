
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

如果未指定任何参数，则检查相应的环境变量（参见“环境变量”一节）。如果环境变量也未设置，则使用硬连接默认值。返回值是指向表示后端连接的抽象结构的指针。


## 2.2 PQsetdbLogin()  
建立与数据库服务器的新连接。其函数原型如下：
```c
PGconn *PQsetdbLogin(const char *pghost, const char *pgport,
							const char *pgoptions, const char *pgtty,
							const char *dbName,
							const char *login, const char *pwd);
```

这是具有固定数量的参数但功能相同的PQconnectdb的前身。 

##2.3 PQsetdb()   
建立与数据库服务器的新连接。其函数原型如下：   

```c
#define PQsetdb(M_PGHOST,M_PGPORT,M_PGOPT,M_PGTTY,M_DBNAME)  \
	PQsetdbLogin(M_PGHOST, M_PGPORT, M_PGOPT, M_PGTTY, M_DBNAME, NULL, NULL)
```
这是一个PQsetdbLogin()使用null指针调用login和pwd参数的宏。提供它是为了与非常老的程序向后兼容。  

## 2.4 PQconnectStart()和PQconnectPoll()  
以非阻塞方式建立与数据库服务器的连接。  这两个函数的原型如下：  
```c
PGconn *PQconnectStart(const char *conninfo);
PostgresPollingStatusType PQconnectPoll(PGconn *conn);
```

这两个功能用于打开与数据库服务器的连接，以使您的应用程序执行线程在执行此操作时不会在远程I/O上被阻止。这种方法的重点是，等待I/O完成可以在应用程序的主循环中进行，而不是在内部进行 PQconnectdb()，因此应用程序可以与其他活动并行地管理此操作。  

使用从字符串conninfo中获取的参数进行数据库连接，该参数传递给PQconnectStart()。该字符串的格式与上面针对PQconnectdb()所述的格式相同。  
- PQconnectStart和PQconnectPoll都不会阻塞，只要满足一定数量的限制:   
    - 使用hostaddr和host参数可以确保不进行名称和反向名称查询。有关详细信息，请参见上面PQconnectdb()下这些参数的文档。  
    - 如果调用PQtrace，请确保跟踪到的流对象不会阻塞。   
    - 在调用PQconnectPoll()之前，请确保套接字处于合适的状态，如下所述。   

要开始非阻塞连接请求，请调用`conn = PQconnectStart("connection_info_string")`。如果`conn`为`null`，则libpq无法分配新的`PGconn` 结构。否则，将返回有效的`PGconn`指针（尽管尚未表示与数据库的有效连接）。从`PQconnectStart()`返回时，调用`status = PQstatus(conn)`。如果status等于`CONNECTION_BAD`，则`PQconnectStart()`失败。  

如果`PQconnectStart()`成功，下一阶段是轮询`libpq`，以便它可以继续连接序列。使用PQsocket(conn)获取数据库连接底层套接字的描述符。这样循环：如果`PQconnectPoll(conn)`最后返回`PGRES_POLLING_READING`，等待直到套接字准备读取(通过`select()`， `poll()`或类似的系统函数表示)。然后再次调用`PQconnectPoll(conn)`。相反，如果`PQconnectPoll(conn)`最后返回`pgres_polling_write`，等待套接字准备写入，然后再次调用PQconnectPoll(conn)。如果你还没有调用PQconnectPoll，即，刚刚调用`PQconnectStart()`，表现为它最后返回`PGRES_POLLING_WRITING`。继续这个循环，直到`PQconnectPoll(conn)`返回`PGRES_POLLING_FAILED`，表示连接过程失败，或者返回`PGRES_POLLING_OK`，表示连接已经成功建立。   

在连接期间的任何时候，都可以通过调用`PQstatu()`来检查连接的状态。如果给出`CONNECTION_BAD`，则连接过程失败;如果它给出`CONNECTION_OK`，那么连接就准备好了。上述`PQconnectPoll()`的返回值同样可以检测到这两种状态。其他状态也可能发生在(且仅发生在)异步连接过程中。这些指示连接过程的当前阶段，例如，向用户提供反馈可能是有用的。这些状态是：






