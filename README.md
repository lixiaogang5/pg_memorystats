
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

执行postgres时候，如果选项参数是“--single”，则走PostgresMain()分支，该分支也称为“单用户模式”。single模式通常会在以下场景中使用：
（1）initdb节点  
（2）多用户模式无法工作时，可使用单用户模式连接、登录数据库  
（3）修复系统故障等。  

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


## 2.1 检查pg_contrl文件中checkPoint内容有效性
下面列出的代码部分是StartupXLOG()函数中对于上面提到的五个功能点的代码逻辑实现：

```c
void
StartupXLOG(void)
{
	XLogCtlInsert *Insert;
	CheckPoint	checkPoint;
	bool		wasShutdown;
	bool		reachedRecoveryTarget 	= false;
	bool		haveBackupLabel 		= false;
	bool		haveTblspcMap 			= false;
	XLogRecPtr	RecPtr,
				checkPointLoc,
				EndOfLog;
	TimeLineID	EndOfLogTLI;
	TimeLineID	PrevTimeLineID;
	XLogRecord *record;
	TransactionId oldestActiveXID;
	bool		backupEndRequired = false;
	bool		backupFromStandby = false;
	DBState		dbstate_at_startup;
	XLogReaderState *xlogreader;
	XLogPageReadPrivate private;
	bool		fast_promoted = false;
	struct stat st;

	// 1. 我们应该有一个辅助进程资源所有者来使用，并且我们不应该处于安装了其他资源所有者的事务中。
	Assert(AuxProcessResourceOwner != NULL);
	Assert(CurrentResourceOwner == NULL ||
		   CurrentResourceOwner == AuxProcessResourceOwner);
	CurrentResourceOwner = AuxProcessResourceOwner;

	// 2. 检查内容是否有效。
	if (!XRecOffIsValid(ControlFile->checkPoint))
		ereport(FATAL,
				(errmsg("control file contains invalid checkpoint location")));

	switch (ControlFile->state)
	{
		case DB_SHUTDOWNED:

			/*
			 * This is the expected case, so don't be chatty in standalone
			 * mode
			 */
			ereport(IsPostmasterEnvironment ? LOG : NOTICE,
					(errmsg("database system was shut down at %s",
							str_time(ControlFile->time))));
			break;

		case DB_SHUTDOWNED_IN_RECOVERY:
			ereport(LOG,
					(errmsg("database system was shut down in recovery at %s",
							str_time(ControlFile->time))));
			break;

		case DB_SHUTDOWNING:
			ereport(LOG,
					(errmsg("database system shutdown was interrupted; last known up at %s",
							str_time(ControlFile->time))));
			break;

		case DB_IN_CRASH_RECOVERY:
			ereport(LOG,
					(errmsg("database system was interrupted while in recovery at %s",
							str_time(ControlFile->time)),
					 errhint("This probably means that some data is corrupted and"
							 " you will have to use the last backup for recovery.")));
			break;

		case DB_IN_ARCHIVE_RECOVERY:
			ereport(LOG,
					(errmsg("database system was interrupted while in recovery at log time %s",
							str_time(ControlFile->checkPointCopy.time)),
					 errhint("If this has occurred more than once some data might be corrupted"
							 " and you might need to choose an earlier recovery target.")));
			break;

		case DB_IN_PRODUCTION:
			ereport(LOG,
					(errmsg("database system was interrupted; last known up at %s",
							str_time(ControlFile->time))));
			break;

		default:
			ereport(FATAL,
					(errmsg("control file contains invalid database cluster state")));
	}

	/* This is just to allow attaching to startup process with a debugger */
#ifdef XLOG_REPLAY_DELAY
	if (ControlFile->state != DB_SHUTDOWNED)
		pg_usleep(60000000L);
#endif

	/*
	 * 3. 请验证pg_wal和pg_wal/archive_status是否存在。如果有人为PITR执行了复制，
	 *    这些目录可能已被排除，需要重新创建。
	 */
	ValidateXLOGDirectoryStructure();



	/*----------
	 * If we previously crashed, perform a couple of actions:
	 如果我们之前崩溃了，执行以下几个动作:
	 *
	 * - The pg_wal directory may still include some temporary WAL segments
	 *   used when creating a new segment, so perform some clean up to not
	 *   bloat this path.  This is done first as there is no point to sync
	 *   this temporary data.
	 pg_wal目录可能仍然包含一些创建新段时使用的临时WAL段，因此执行一些清理操作以避免该路径膨胀。
	 首先执行这个操作，因为没有必要同步此临时数据。
	 *
	 * - There might be data which we had written, intending to fsync it, but
	 *   which we had not actually fsync'd yet.  Therefore, a power failure in
	 *   the near future might cause earlier unflushed writes to be lost, even
	 *   though more recent data written to disk from here on would be
	 *   persisted.  To avoid that, fsync the entire data directory.
	 可能有一些我们已经写入的数据，打算进行fsync，但实际上还没有进行fsync。
	 因此，在不久的将来发生电源故障可能会导致早期未刷新的写操作丢失，
	 即使从这里开始写入磁盘的最新数据将被持久化。为了避免这种情况，fsync会同步整个数据目录。
	 */

	// 使用kill -9 可复现此场景.  ---->      pg_control
	// Database cluster state:               in production
	if (ControlFile->state != DB_SHUTDOWNED &&
		ControlFile->state != DB_SHUTDOWNED_IN_RECOVERY)			
	{
		RemoveTempXlogFiles();
		SyncDataDirectory();
	}
    
    ...... // 省略若干部分
}
```

首先，CurrentResourceOwner和AuxProcessResourceOwner这两个全局变量在StartupXLOG()函数的前面已经初始化了。由InitPostgres()函数内部的CreateAuxProcessResourceOwner()函数完成，该函数完成了这里两个资源初始化之后，才紧接着调用StartupXLOG()函数。

![10](https://user-images.githubusercontent.com/63132178/181516202-28e65704-ed1e-423b-9f96-b2dd3d97fbd3.png)

下面开始检查ControlFile->checkPoint变量值的内容是否正确。ControlFile是一个static的全局指针变量，该变量中的内容在PostmasterMain()函数中通过LocalProcessControlFile(false)负责初始化。它主要是通过读取PGDATA路径下global/目录下的pg_control文件初始化。所以在启动postgres之前，我们必须对该文件中的checkPoint的值的有效性进行判断处理，如果满足条件，则继续下面的执行；反之则结束任务。

检查ControlFile->checkPoint内容的有效性由XRecOffIsValid()完成，它是一个宏，其定义如下：

```c
typedef struct XLogPageHeaderData
{
	uint16		xlp_magic;		/* magic value for correctness checks */
	uint16		xlp_info;		/* flag bits, see below */
	TimeLineID	xlp_tli;		/* TimeLineID of first record on page */
	XLogRecPtr	xlp_pageaddr;	/* XLOG address of this page */

	/*
	 * When there is not enough space on current page for whole record, we
	 * continue on the next page.  xlp_rem_len is the number of bytes
	 * remaining from a previous page.
	 *
	 * Note that xlp_rem_len includes backup-block data; that is, it tracks
	 * xl_tot_len not xl_len in the initial header.  Also note that the
	 * continuation data isn't necessarily aligned.
	 */
	uint32		xlp_rem_len;	/* total len of remaining data for record */
} XLogPageHeaderData;

// SizeOfXLogShortPHD: 24
#define SizeOfXLogShortPHD	MAXALIGN(sizeof(XLogPageHeaderData))

#define XLOG_BLCKSZ 8192

// 检查XLogRecPtr值是否在合理范围内
#define XRecOffIsValid(xlrp) \
		((xlrp) % XLOG_BLCKSZ >= SizeOfXLogShortPHD)
```

其中SizeOfXLogShortPHD是一个宏，其等价于MAXALIGN(sizeof(XLogPageHeaderData))，值为24字节，宏XLOG_BLCKSZ的大小是8192字节。因此若 ControlFile->checkPoint % 8192 >= 24，则表明该checkPoint内容是不正确的，从而结束postgres服务的启动。
![image](https://user-images.githubusercontent.com/63132178/181519499-2a7f14a5-726e-4852-ab65-ed6dc4977972.png)

## 2.2 日志打印上一次postgres集群的状态
若ControlFile->checkPoint的内容正确，则接着对postgres集群服务上一次关闭时候的状态进行日志打印提示。这部分功能如下：

![image](https://user-images.githubusercontent.com/63132178/181520330-d2bff0f8-1ba9-4715-a124-21df50dc30a9.png)

其对应的代码如下：  
```c
switch (ControlFile->state)
	{
		case DB_SHUTDOWNED:

			/*
			 * This is the expected case, so don't be chatty in standalone
			 * mode
			 */
			ereport(IsPostmasterEnvironment ? LOG : NOTICE,
					(errmsg("database system was shut down at %s",
							str_time(ControlFile->time))));
			break;

		case DB_SHUTDOWNED_IN_RECOVERY:
			ereport(LOG,
					(errmsg("database system was shut down in recovery at %s",
							str_time(ControlFile->time))));
			break;

		case DB_SHUTDOWNING:
			ereport(LOG,
					(errmsg("database system shutdown was interrupted; last known up at %s",
							str_time(ControlFile->time))));
			break;

		case DB_IN_CRASH_RECOVERY:
			ereport(LOG,
					(errmsg("database system was interrupted while in recovery at %s",
							str_time(ControlFile->time)),
					 errhint("This probably means that some data is corrupted and"
							 " you will have to use the last backup for recovery.")));
			break;

		case DB_IN_ARCHIVE_RECOVERY:
			ereport(LOG,
					(errmsg("database system was interrupted while in recovery at log time %s",
							str_time(ControlFile->checkPointCopy.time)),
					 errhint("If this has occurred more than once some data might be corrupted"
							 " and you might need to choose an earlier recovery target.")));
			break;

		case DB_IN_PRODUCTION:
			ereport(LOG,
					(errmsg("database system was interrupted; last known up at %s",
							str_time(ControlFile->time))));
			break;

		default:
			ereport(FATAL,
					(errmsg("control file contains invalid database cluster state")));
	}
```
ControlFile->state中state成员是一个DBState枚举类型，其声明如下：
```c
/*
 * System status indicator.  Note this is stored in pg_control; if you change
 * it, you must bump PG_CONTROL_VERSION
 */
typedef enum DBState
{
	DB_STARTUP = 0,
	DB_SHUTDOWNED,
	DB_SHUTDOWNED_IN_RECOVERY,
	DB_SHUTDOWNING,
	DB_IN_CRASH_RECOVERY,
	DB_IN_ARCHIVE_RECOVERY,
	DB_IN_PRODUCTION
} DBState;
```
从pg_control文件中可以看到，postgres集群上一次停掉服务后的记录状态是DB_SHUTDOWNED（对应shut down），这个枚举值与字符串直接的转换参考函数dbState()。

```c
tatic const char *
dbState(DBState state)
{
	switch (state)
	{
		case DB_STARTUP:
			return _("starting up");
		case DB_SHUTDOWNED:
			return _("shut down");
		case DB_SHUTDOWNED_IN_RECOVERY:
			return _("shut down in recovery");
		case DB_SHUTDOWNING:
			return _("shutting down");
		case DB_IN_CRASH_RECOVERY:
			return _("in crash recovery");
		case DB_IN_ARCHIVE_RECOVERY:
			return _("in archive recovery");
		case DB_IN_PRODUCTION:
			return _("in production");
	}
	return _("unrecognized status code");
}
```

## 2.3 验证pg_wal/archive_status是否存在
接下来就是去验证pg_wal和pg_wal/archive_status是否存在。如果有人为PITR执行了复制，这些目录可能已被排除，需要重新创建。pg_wal目录位于PGDATA路径下，而archive_status则位于pg_wal目录中。
![image](https://user-images.githubusercontent.com/63132178/181677875-78908c45-20b6-4847-ac0c-e8da25ef41c4.png)

此部分内容主要是由函数ValidateXLOGDirectoryStructure()负责。该函数的实现如下所示：
```c
static void
ValidateXLOGDirectoryStructure(void)
{
	char		path[MAXPGPATH];
	struct stat stat_buf;

	// 检查pg_wal;如果不存在，则出错
	if (stat(XLOGDIR, &stat_buf) != 0 ||
		!S_ISDIR(stat_buf.st_mode))
		ereport(FATAL,
				(errmsg("required WAL directory \"%s\" does not exist",
						XLOGDIR)));

	/* Check for archive_status */
	snprintf(path, MAXPGPATH, XLOGDIR "/archive_status");
	if (stat(path, &stat_buf) == 0)
	{
		/* Check for weird cases where it exists but isn't a directory */
		if (!S_ISDIR(stat_buf.st_mode))
			ereport(FATAL,
					(errmsg("required WAL directory \"%s\" does not exist",
							path)));
	}
	else
	{
		ereport(LOG,
				(errmsg("creating missing WAL directory \"%s\"", path)));
		if (MakePGDirectory(path) < 0)
			ereport(FATAL,
					(errmsg("could not create missing directory \"%s\": %m",
							path)));
	}
}

```

校验pg_wal和pg_wal/archive_status是否存在。如果后者不存在，则重新创建它。这个函数的目标不是验证这些目录的内容，而是在有人出于PITR目的执行了集群复制，但从复制中省略了pg_wal的情况下提供帮助。如果pg_wal不存在，我们也可以重新创建它，但我们经过深思熟虑后决定不创建。pg_wal作为符号链接是很常见的，如果这是DBA的意图，那么自动创建普通目录将导致性能下降，而不会引起注意

该函数实现中，XLOGDIR是一个宏，其值为“pg_wal”，该宏声明于xlog_internal.h头文件中。
```c
/*
 * The XLog directory and control file (relative to $PGDATA)
 */
#define XLOGDIR				"pg_wal"
#define XLOG_CONTROL_FILE	"global/pg_control"
```




















