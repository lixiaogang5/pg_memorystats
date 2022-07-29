
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

	/* 3. 请验证pg_wal和pg_wal/archive_status是否存在。如果有人为PITR执行了复制，
	 *    这些目录可能已被排除，需要重新创建。
	 */
	ValidateXLOGDirectoryStructure();

	// 4. 如果上一次postgres服务奔溃，则需要依次执行下面两个函数
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
如果pg_wal目录不存在或者是pg_wal不是一个目录文件，则打印相关错误日志信息，并结束postgres进程的初始化流程。如果pg_wal存储且是一个目录文件，则继续对pg_wal目录下的archive_status目录文件进行判断，若该文件存在，但不是目录文件，则报错并结束postgres守护进程的初始化任务。反之，若archive_status目录文件不存在，则日志打印提示信息，并且postgres负责创建一个空的archive_status目录文件。可总结得出下面两点：  
（1） 对于pg_wal文件，必须满足两个要求，首先它必须存在，其次，它还必须是一个目录文件；   
（2） archive_status目录文件需满足的要求是：它可以不存在，若不存在，postgres进程会重新创建一个；但是若它存在，则它必须是目录文件，否则，postgres也会启动失败。


## 2.4 判断postgres服务上一次的是否处于奔溃

如果我们之前崩溃了，执行以下几个动作:      
（1）pg_wal目录可能仍然包含一些创建新段时使用的临时WAL段，因此执行一些清理操作以避免该路径膨胀。 首先执行这个操作，因为没有必要同步此临时数据。     
（2） 可能有一些我们已经写入的数据，打算进行fsync，但实际上还没有进行fsync。 因此，在不久的将来发生电源故障可能会导致早期未刷新的写操作丢失， 即使从这里开始写入磁盘的最新数据将被持久化。为了避免这种情况，fsync会同步整个数据目录。

此部分的逻辑代码如下：
```c
// 4. 如果上一次postgres服务奔溃，则需要依次执行下面两个函数
// 使用kill -9 可复现此场景.  ---->      pg_control
// Database cluster state:               in production

if (ControlFile->state 	!= DB_SHUTDOWNED &&
    ControlFile->state 	!= DB_SHUTDOWNED_IN_RECOVERY)			
{
	RemoveTempXlogFiles();
	SyncDataDirectory();
}
```
若上一次postgres服务为正常停止、关闭，则位于pg_control文件中的state字段状态值为“shut down（DB_SHUTDOWNED）”，此情况下不会进入到if(){}语句的函数部分；反之若上一次的postgres为异常终止，比如postgres守护进程运行过程中，我在另外一个终端中通过kill -9 PID方式来kill掉postgres服务，那么pg_control文件中的state字段就不是DB_SHUTDOWNED状态，则会进入到if(){}的函数体中。然后依次执行RemoveTempXlogFiles()和SyncDataDirectory()函数。

如下图所示，左上方图1是postgres启动初始化过程；左下方图2通过kill -9 PID杀掉postgres进程，右上方图1是postgrs进程被kill掉后，通过pg_controldata程序得到的pg_control文件中集群的state，可以看到该状态是postgres被kill掉之前的集群状态，即in production。

![image](https://user-images.githubusercontent.com/63132178/181693518-87e47f8d-dd53-4407-9ddf-1afe75efa75f.png)



### 2.4.1 移除pg_wal下的临时WAL段
 删除pg_wal中的所有临时日志文件，这在上次崩溃后的恢复开始时调用，此时没有其他进程写入新的WAL数据。该功能由函数RemoveTempXlogFiles()完成，其函数完整实现如下：
 ```c
 static void
RemoveTempXlogFiles(void)
{
	DIR		   *xldir;
	struct dirent 	   *xlde;

	elog(DEBUG2, "removing all temporary WAL segments");

	// 指定要打开的目录（XLOGDIR = “pg_wal”）， AllocateDir()函数底层封装了opendir()系统函数
	xldir = AllocateDir(XLOGDIR);
	// 循环读取pg_wal目录下的所有文件（包括目录），凡是以xlogtemp.开头的文件都表示临时文件，都将一一删除
	while ((xlde = ReadDir(xldir, XLOGDIR)) != NULL)
	{
		char		path[MAXPGPATH];

		if (strncmp(xlde->d_name, "xlogtemp.", 9) != 0)
			continue;
		// 在读取pg_control文件时候，已经chdir()到PGDATA， 这里拼接临时文件的绝对路径.
		snprintf(path, MAXPGPATH, XLOGDIR "/%s", xlde->d_name);
		// 调用unlink()系统函数删除该文件.
		unlink(path);
		// 日志打印已移除的文件
		elog(DEBUG2, "removed temporary WAL segment \"%s\"", path);
	}
	FreeDir(xldir);
}
 ```
函数AllocateDir()底层封装了opendir()系统函数，如下：
```c
DIR *
AllocateDir(const char *dirname)
{
	DIR		   *dir;

	DO_DB(elog(LOG, "AllocateDir: Allocated %d (%s)",
			   numAllocatedDescs, dirname));

	/* Can we allocate another non-virtual FD? */
	if (!reserveAllocatedDesc())
		ereport(ERROR,
				(errcode(ERRCODE_INSUFFICIENT_RESOURCES),
				 errmsg("exceeded maxAllocatedDescs (%d) while trying to open directory \"%s\"",
						maxAllocatedDescs, dirname)));

	/* Close excess kernel FDs. */
	ReleaseLruFiles();

TryAgain:
	if ((dir = opendir(dirname)) != NULL)
	{
		AllocateDesc *desc = &allocatedDescs[numAllocatedDescs];

		desc->kind 			= AllocateDescDir;
		desc->desc.dir 		= dir;
		desc->create_subid 	= GetCurrentSubTransactionId();
		numAllocatedDescs++;
		return desc->desc.dir;
	}

	if (errno == EMFILE || errno == ENFILE)
	{
		int			save_errno = errno;

		ereport(LOG,
				(errcode(ERRCODE_INSUFFICIENT_RESOURCES),
				 errmsg("out of file descriptors: %m; release and retry")));
		errno = 0;
		if (ReleaseLruFile())
			goto TryAgain;
		errno = save_errno;
	}

	return NULL;
}
```
函数ReadDir()底层封装了readdir()函数。

### 2.4.2 fsync() PGDATA目录下的所有文件
在PGDATA及其所有内容上递归发出fsync。我们对常规文件和目录进行fsync,无论它们在哪里，但我们只关注pg_wal和紧接在pg_tblspc下的符号链接。其他符号链接被假定指向我们不负责fsyncing的文件，并且可能根本没有写入权限。错误会被记录，但不会被认为是致命的;这是因为它仅在数据库启动期间使用，以处理数据目录中存在已发出但未同步的写入挂起的可能性。我们希望确保这样的写操作在新运行中执行的任何操作之前到达磁盘。但是，对于无害的情况(比如数据目录中的只读文件)，在出错时中止将导致启动失败，这也不好。

注意，如果我们之前因为fsync()上的PANIC而崩溃，那么我们将在恢复期间再次重写所有更改。 注意，我们假设我们一开始就被chdir到PGDATA（这个在PostmasterMain()函数中读取pg_control文件的时候已经chdir到PGDATA下了）。

这个过程由函数SyncDataDirectory()完成，其实现如下：
```c
void
SyncDataDirectory(void)
{
	bool		xlog_is_symlink;

	// 1. 如果fsync被禁用，我们可以跳过这整个过程。
	if (!enableFsync)
		return;

	// 2. 如果pg_wal是一个符号链接，我们需要单独递归到它，因为下面的第一个walkdir将忽略它。
	xlog_is_symlink = false;

#ifndef WIN32
	{
		struct stat st;
		
		//lstat()系统函数打开一个符号链接文件
		if (lstat("pg_wal", &st) < 0)
			ereport(LOG,
					(errcode_for_file_access(),
					 errmsg("could not stat file \"%s\": %m",
							"pg_wal")));
		else if (S_ISLNK(st.st_mode))
			xlog_is_symlink = true;
	}
#else
	if (pgwin32_is_junction("pg_wal"))
		xlog_is_symlink = true;
#endif

	// 3.  如果可能的话，向内核提示我们将很快对数据目录及其内容进行fsync。
	//     这个步骤中的错误甚至比正常情况下更无趣，所以只在DEBUG1中记录它们。
#ifdef PG_FLUSH_DATA_WORKS
	walkdir(".", pre_sync_fname, false, DEBUG1);
	if (xlog_is_symlink)
		walkdir("pg_wal", pre_sync_fname, false, DEBUG1);
	walkdir("pg_tblspc", pre_sync_fname, true, DEBUG1);
#endif
	
	/* 4. 现在我们以相同的顺序执行fsync()。
	 *    主调用忽略符号链接，因此，如果pg_wal是符号链接，除了特别处理它之外，
	 *    还必须使用process_symlinks = true单独访问pg_tblspc。注意，如果pg_tblspc中有任何普通目录，
	 *    它们将被fsync两次。这不是一个预期情况，所以我们不用担心优化。
	 */
	// 由于当前处于PGDATA路径下，执行“.”，则表示将PGDATA目录下的所有文件/目录（递归）进行fsync()
	walkdir(".", datadir_fsync_fname, false, LOG);
	if (xlog_is_symlink)
		walkdir("pg_wal", datadir_fsync_fname, false, LOG);
		
	// 对pg_tblspc目录下的所有文件/目录执行fsync()
	walkdir("pg_tblspc", datadir_fsync_fname, true, LOG);
}
```
walkdir()函数中的第二个参数pre_sync_fname是一个函数指针，walkdir()的实现如下：
```c
static void
walkdir(const char *path,
		void (*action) (const char *fname, bool isdir, int elevel),
		bool process_symlinks,
		int elevel)
{
	DIR		   *dir;
	struct dirent *de;
	
	// 打开指定的目录文件
	dir = AllocateDir(path);
	while ((de = ReadDirExtended(dir, path, elevel)) != NULL)
	{
		char		subpath[MAXPGPATH * 2];
		struct stat fst;
		int			sret;

		CHECK_FOR_INTERRUPTS();		////

		// 忽略“.”和“..”这两个特殊的目录
		if (strcmp(de->d_name, ".") == 0 ||
			strcmp(de->d_name, "..") == 0)
			continue;

		snprintf(subpath, sizeof(subpath), "%s/%s", path, de->d_name);

		if (process_symlinks)
			sret = stat(subpath, &fst);
		else
			sret = lstat(subpath, &fst);

		if (sret < 0)
		{
			ereport(elevel,
					(errcode_for_file_access(),
					 errmsg("could not stat file \"%s\": %m", subpath)));
			continue;
		}
		
		// 如果是普通文件，则调用pre_sync_fname()函数，
		if (S_ISREG(fst.st_mode))
			(*action) (subpath, false, elevel);
			
		// 反之，若当前文件是一个目录，则递归遍历
		else if (S_ISDIR(fst.st_mode))
			walkdir(subpath, action, false, elevel);
	}

	FreeDir(dir);	/* we ignore any error here */

	 // 对目标目录本身进行fsync是很重要的，因为单个文件fsync不能保证文件的目录条目是同步的。
	 // 但是，如果AllocateDir失败，则跳过此操作;action函数对此可能并不稳健。
	if (dir)
		(*action) (path, true, elevel);
}

```
函数Walkdir()的作用是递归遍历一个目录，将该操作应用到每个常规文件和目录(包括已命名目录本身)。如果process_symlinks为真，则动作和递归也应用于给定目录中符号链接指向的普通文件和目录；否则将忽略符号链接。符号链接在子目录中总是被忽略的，即我们故意不将process_symlinks标志传递给递归调用。错误报告级别为elevel，可能是ERROR或更少。
函数pre_sync_fname()的作用是打开该文件，得到文件对应的文件句柄fd，然后再该函数内部通过间接调用pg_flush_data()函数，从而把这个文件句柄fd对应的数据全部刷新到磁盘上面。

# 3. 总结
本文详细讲解了postgres服务启动的大致流程，同时也对XLOG的启动机制进行了大致的描述，比如会对pg_control文件中的checkPoint、state等字段值进行判断、校验，然后又对pg_wal文件和archive_status目录文件进行判断，从而决定下一步的操作。后面又提到了如果上一次的postgres进程是非正常手段关机，则本次的postgres服务启动将会执行系列的初始化操作，比如删除pg_wal目录下的所有临时文件，fsync() PGDATA目录下的所有文件，将其刷到磁盘持久化存储。




