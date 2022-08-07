## 问题
在项目调试过程中，有同学遇到一个crash问题，控制台log：
```
BUG IN CLIENT OF libsqlite3.dylib: illegal multi-threaded access to database connection
```
应该“庆幸”这个问题在调试的过程中复现了，很明显是一个多线程操作数据库问题，如果是线上崩溃日志呢，我截取了部分堆栈：
```
Trigger Thread:18
appversion : 13.12.1.10
CFBundleShortVersionString : 1.1.0
CFBundleName : 测试
packagename : com.test.DemoProject
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
9    DemoProject                      -[FMDatabase executeQuery:]	(in DemoProject)	(FMDatabase.m:0)	24
10   DemoProject                      ___37-[DemoAssetManager demo_cleanAssets]_block_invoke	(in DemoProject)	(DemoAssetManager.m:81)	0
11   libdispatch.dylib                __dispatch_call_block_and_release	(in libdispatch.dylib)	24
12   libdispatch.dylib                __dispatch_client_callout	(in libdispatch.dylib)	16
13   libdispatch.dylib                __dispatch_lane_serial_drain$VARIANT$mp	(in libdispatch.dylib)	612
14   libdispatch.dylib                __dispatch_lane_invoke$VARIANT$mp	(in libdispatch.dylib)	420
15   libdispatch.dylib                __dispatch_workloop_worker_thread	(in libdispatch.dylib)	712
16   libsystem_pthread.dylib          __pthread_wqthread	(in libsystem_pthread.dylib)	272
```
可以看到是**BUS_ADRALN**信号量崩溃，堆栈指向数据库查找操作，如果没有查到对应其他线程操作数据库，很容易认为没有多线程操作而忽略多线程访问问题。如果没有明确的问题指向性，我们不访从数据库基础来分析下。
## 分析
子线程信号量崩溃，无外乎内存问题，对应野指针、多线程资源竞争等，那就先查查多线程呗，思考一个问题：sqlite是否是线程安全的？
带着问题在官方文档看到这样一个问答 [Is SQLite threadsafe?](https://www.sqlite.org/faq.html#q6)，大致意思是说：
> SQLite是线程安全的。但是为了保证线程安全，SQLite必须在编译的时候将预处理的宏 SQLITE_THREADSAFE 设置为1。如果你不确定链接的SQLite二进制是否线程安全的，可以通过调用[sqlite3_threadsafe()](https://www.sqlite.org/c3ref/threadsafe.html)来确认。
> SQLite是线程安全的，因为它使用了互斥量来串行访问通用数据结构。但是频繁的获取和释放互斥量会降低SQLite的性能。如果你不需要保证线程安全，应该禁用互斥量来最大化性能。可以参考[threading mode](https://www.sqlite.org/threadsafe.html)获取更多的信息。

嗯，看完还是有点模棱两可，说是线程安全但是需要编译时设置预处理宏，我们接着看上面提到的[threading mode](https://www.sqlite.org/threadsafe.html)：
> SQLite支持3种不同的线程模式：
> 1. 单线程模式。所有的互斥量会被禁用，且SQLite在多个线程模式下是不安全的。
> 2. 多线程模式。在没有多线程同时访问同一个数据库连接时，SQLite是线程安全的。可以理解为：多线程可以同时访问不同的数据库连接，但是不可以同时访问同一个数据库连接。
> 3. 串行模式。 没有任何限制，SQlite操作是线程安全的。
那么对于iOS集成的libsqlite3.dylib配置的是哪个参数呢，调用如下函数：
```
// mode = 2
// 该方法返回编译时对应配置项值，和运行时改变配置项无关
int mode = sqlite3_threadsafe(); 
```
由上面输出可以得出，iOS链接的libsqlite3.dylib配置的是 "多线程模式"，那么何为数据库连接呢？
数据库文件，打开、关闭、增、删、改、查操作，都称为数据库连接。
结合文档“多线程模式”可以知道：同一个数据库文件操作必须是串行的，否则就会出现线程安全问题。

## 验证
我们使用FMDB进行验证:
```
@interface AssetManager ()

@property (nonatomic, strong) FMDatabase *database;

@end

@implementation AssetManager

#pragma mark public
- (void)testMultiThread {
    // 多线程
    for (int i = 0; i < 1000; i ++) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
           [self.database executeQuery:@"select * from download where a = '1'"];
        });
    }
    
    for (int i = 0; i < 1000; i ++) {
        [self.database executeQuery:@"select * from download where a = '1'"];
    }
}
#pragma mark - private 
- (FMDatabase *)database {
    if (!_database) {
        FMDatabase *database = [FMDatabase databaseWithPath:[[self rootDirectory] stringByAppendingPathComponent:@"assets.db"]];
        if ([database open]) {
            [database executeUpdate:@"create table if not exists download (a text, b text, c text)"];
            self.database = database;
        }
    }
    return _database;
}

- (NSString *)rootDirectory {
    NSString *documentPath = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).firstObject;
    if (!documentPath) {
        return nil;
    }
    // 获取文件夹路径
    NSString *rootDir = [documentPath stringByAppendingPathComponent:@"asset_service"];
    if (![NSFileManager.defaultManager fileExistsAtPath:rootDir]) {
        NSError *error;
        [NSFileManager.defaultManager createDirectoryAtPath:rootDir withIntermediateDirectories:YES attributes:nil error:&error];
        if (error) {
            NSAssert(NO, @"ABTest: creat directory failure.");
            return nil;
        }
    }
    return rootDir;
}
@end
```
如上测试，多线程查询数据库会导致crash，所以一定要保证数据库操作的线程同步。
> 2022-08-07 完结FMDB多线程分析，不是太深入，只是结合原理对多线程初步分析，后续项目中遇到其他问题，再进行完善。
