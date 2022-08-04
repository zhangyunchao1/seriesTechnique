在项目调试过程中，有同学遇到一个crash问题，控制台log：
```
BUG IN CLIENT OF libsqlite3.dylib: illegal multi-threaded access to database connection
```
应该“庆幸”这个问题在调试的过程中复现了，很明显是一个多线程操作数据库问题，如果是线上崩溃日志呢，我截取了部分堆栈：
```
Trigger Thread:18
appversion : 13.12.1.10
CFBundleShortVersionString : 1.1.0
CFBundleName : 百度
packagename : com.baidu.DemoProject
boot_time : 2022-07-07T20:55:22Z
process_name : DemoProject
process_id : 6820
parent_process_id : 1
Exception Codes: BUS_ADRALN at 0x0000004000006f88
Exception Type: SIGBUS
ExtraInfo:{FirstLaunch:1,FirstInstall:0,remjsVer = 3.310.31, isPrejs = 1, prejsVer = 3.490.5, }
Code Type: arm64
OS Version: iPhone OS (18F72)
Launch Time: 2022-07-08 15:02:41
Date/Time: 2022-07-08 15:02:45

Thread 18 Crashed:
0    libsqlite3.dylib                 _sqlite3_sourceid	(in libsqlite3.dylib)	1576
1    libsqlite3.dylib                 _sqlite3_log	(in libsqlite3.dylib)	44728
2    libsqlite3.dylib                 _sqlite3_sourceid	(in libsqlite3.dylib)	157664
3    libsqlite3.dylib                 _sqlite3_log	(in libsqlite3.dylib)	7900
4    libsqlite3.dylib                 _sqlite3_exec	(in libsqlite3.dylib)	11068
5    libsqlite3.dylib                 _sqlite3_exec	(in libsqlite3.dylib)	5704
6    libsqlite3.dylib                 _sqlite3_exec	(in libsqlite3.dylib)	3228
7    libsqlite3.dylib                 _sqlite3_exec	(in libsqlite3.dylib)	2376
8    DemoProject                      -[FMDatabase executeQuery:withArgumentsInArray:orDictionary:orVAList:]	(in DemoProject)	(FMDatabase.m:836)	0
9    DemoProject                      -[FMDatabase executeQuery:]	(in BaiduBoxApp)	(FMDatabase.m:0)	24
10   DemoProject                      ___37-[DemoAssetManager demo_cleanAssets]_block_invoke	(in DemoProject)	(DemoAssetManager.m:81)	0
11   libdispatch.dylib                __dispatch_call_block_and_release	(in libdispatch.dylib)	24
12   libdispatch.dylib                __dispatch_client_callout	(in libdispatch.dylib)	16
13   libdispatch.dylib                __dispatch_lane_serial_drain$VARIANT$mp	(in libdispatch.dylib)	612
14   libdispatch.dylib                __dispatch_lane_invoke$VARIANT$mp	(in libdispatch.dylib)	420
15   libdispatch.dylib                __dispatch_workloop_worker_thread	(in libdispatch.dylib)	712
16   libsystem_pthread.dylib          __pthread_wqthread	(in libsystem_pthread.dylib)	272
```
