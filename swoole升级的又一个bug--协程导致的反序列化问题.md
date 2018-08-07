# 0x01 背景
swoole发布了不少版本了，但是我司用的版本还是比较旧的版本，遇到过不少坑。之前升级也遇到了一些BUG，比如swoole_server不能序列化和反序列化，不过调试并解决了，于是一些服务升级到了2.1.3。但是最近一个服务在swoole升级中出现了一个问题，问题点在onTask回调中发生错误：`Fatal error: Class declarations may not be nested in ...`。也就是说在worker发给task worker的内容反序列化时出现问题，导致task worker异常退出。

这个错误，按照我的理解只会在类嵌套定义时才会出现，类似如下：

```php
<?php 
class MyClass
{
    public function TestMethod()
    {
        class MyNestedClass
        {

        }
    }
}

// Fatal error: Class declarations may not be nested in xxx.php on line 6
```

但查看了报错的代码，没有嵌套定义啊，而且嵌套定义时那是必现的错误，但是升级前没有问题。这个错误是PHP内核抛出的错误，但只升级了swoole，为什么会导致这个错误呢？如果没有修改内核，应该是不会出现这个错误的，后面和跟韩天峰他们沟通的过程中，他们初步猜测应该是协程的问题。下面看看我的分析过程，并且我提供了复现的代码。

# 0x02 问题分析

先说说找这个错误的测试环境：
```shell
# OS
$ uname -a
Linux host-XXX 3.10.0-514.10.2.el7.x86_64 #1 SMP Fri Mar 3 00:04:05 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux

# PHP 
$ php7.0.29-debug -v
PHP 7.0.29 (cli) (built: Jul 23 2018 10:43:25) ( NTS DEBUG )
Copyright (c) 1997-2017 The PHP Group
Zend Engine v3.0.0, Copyright (c) 1998-2017 Zend Technologies

# Swoole
$ php7.0.29-debug --ri swoole

swoole

swoole support => enabled
Version => 2.1.3
Author => tianfeng.han[email: mikan.tenny@gmail.com]
coroutine => enabled
epoll => enabled
eventfd => enabled
timerfd => enabled
signalfd => enabled
cpu affinity => enabled
spinlock => enabled
rwlock => enabled
async http/websocket client => enabled
Linux Native AIO => enabled
pcre => enabled
zlib => enabled
mutex_timedlock => enabled
pthread_barrier => enabled
futex => enabled

Directive => Local Value => Master Value
swoole.aio_thread_num => 2 => 2
swoole.display_errors => On => On
swoole.use_namespace => On => On
swoole.use_shortname => On => On
swoole.fast_serialize => Off => Off
swoole.unixsock_buffer_size => 8388608 => 8388608

# GCC
$ gcc -v
使用内建 specs。
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/lto-wrapper
目标：x86_64-redhat-linux
配置为：../configure --prefix=/usr --mandir=/usr/share/man --infodir=/usr/share/info --with-bugurl=http://bugzilla.redhat.com/bugzilla --enable-bootstrap --enable-shared --enable-threads=posix --enable-checking=release --with-system-zlib --enable-__cxa_atexit --disable-libunwind-exceptions --enable-gnu-unique-object --enable-linker-build-id --with-linker-hash-style=gnu --enable-languages=c,c++,objc,obj-c++,java,fortran,ada,go,lto --enable-plugin --enable-initfini-array --disable-libgcj --with-isl=/builddir/build/BUILD/gcc-4.8.5-20150702/obj-x86_64-redhat-linux/isl-install --with-cloog=/builddir/build/BUILD/gcc-4.8.5-20150702/obj-x86_64-redhat-linux/cloog-install --enable-gnu-indirect-function --with-tune=generic --with-arch_32=x86-64 --build=x86_64-redhat-linux
线程模型：posix
gcc 版本 4.8.5 20150623 (Red Hat 4.8.5-11) (GCC) 

# GDB 
$ gdb -v
GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-94.el7
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
```

第一步先看问题点的出处，搜索发现在php-7.0.29/Zend/zend_compile.c的zend_compile_class_decl函数里：
```c
void zend_compile_class_decl(zend_ast *ast) /* {{{ */
{
    // ...
    if (EXPECTED((decl->flags & ZEND_ACC_ANON_CLASS) == 0)) {
        zend_string *unqualified_name = decl->name;

        if (CG(active_class_entry)) {
            zend_error_noreturn(E_COMPILE_ERROR, "Class declarations may not be nested");
        }
        // ...
    }
}
/* }}} */
```
上面的代码省略了一些代码或者中间过程（使用`// ...`表示），但是不太影响问题分析，后面也这样，我就不重复说明了。从代码可以看出`CG(active_class_entry)`为true，那么就会报错了。这个`CG(active_class_entry)`展开了就是compiler_globals.active_class_entry，compiler_globals.active_class_entry是编译阶段保存当前类的实体信息，对应zend_class_entry结构。也就是在准备编译某个类之前，先判断`CG(active_class_entry)`，不为空说明前面编译的类还没结束，如果再编译现在这个类，那就是类的嵌套定义（匿名类除外，PHP7支持匿名类），PHP语法是不允许这样的情况出现的，所以就报错了。

找内核问题嘛，那就要用到gdb了，上面也说了gdb的版本，ps查看一下task worker进程ID：

![img](https://raw.githubusercontent.com/iam2c/blog/master/assets/swoole_unserialize/4.png?raw=true)

3659-3662都是task worker的进程ID。

挑一个进程进行调试：
```shell
$ gdb attach 3659
GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-94.el7
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
// ...
0x00007fde36e09a93 in __msgrcv_nocancel () from /lib64/libc.so.6
Missing separate debuginfos, use: debuginfo-install glibc-2.17-196.el7_4.2.x86_64 keyutils-libs-1.5.8-3.el7.x86_64 krb5-libs-1.14.1-27.el7_3.x86_64 libcom_err-1.42.9-9.el7.x86_64 libmcrypt-2.5.8-13.el7.x86_64 libselinux-2.5-6.el7.x86_64 libxml2-2.9.1-6.el7_2.3.x86_64 nss-softokn-freebl-3.16.2.3-14.4.el7.x86_64 openssl-libs-1.0.2k-8.el7.x86_64 pcre-8.32-15.el7_2.1.x86_64 xz-libs-5.2.2-1.el7.x86_64 zlib-1.2.7-17.el7.x86_64
(gdb) b /home/www/php-7.0.29/Zend/zend_compile.c:5277
Breakpoint 1 at 0x937b0c: file /home/www/php-7.0.29/Zend/zend_compile.c, line 5277.
(gdb) c
Continuing.
```

发送请求触发问题点，gdb调到断点处：

```shell
(gdb) c
Continuing.

Breakpoint 1, zend_compile_class_decl (ast=0x7fd9b4a4a3d0) at /home/www/php-7.0.29/Zend/zend_compile.c:5277
5277                zend_error_noreturn(E_COMPILE_ERROR, "Class declarations may not be nested");
(gdb) bt
#0  zend_compile_class_decl (ast=0x7fd9b4a4a3d0) at /home/www/php-7.0.29/Zend/zend_compile.c:5277
#1  0x000000000093d701 in zend_compile_stmt (ast=0x7fd9b4a4a3d0) at /home/www/php-7.0.29/Zend/zend_compile.c:7163
#2  0x000000000093d338 in zend_compile_top_stmt (ast=0x7fd9b4a4a3d0)
    at /home/www/php-7.0.29/Zend/zend_compile.c:7069
#3  0x000000000093d31a in zend_compile_top_stmt (ast=0x7fd9b4a4a018)
    at /home/www/php-7.0.29/Zend/zend_compile.c:7064
#4  0x00000000009061a7 in compile_file (file_handle=0x7fffe32348d0, type=2) at Zend/zend_language_scanner.l:608
#5  0x000000000072d353 in phar_compile_file (file_handle=0x7fffe32348d0, type=2)
    at /home/www/php-7.0.29/ext/phar/phar.c:3337
#6  0x00000000009f4c02 in ZEND_INCLUDE_OR_EVAL_SPEC_CV_HANDLER ()
    at /home/www/php-7.0.29/Zend/zend_vm_execute.h:29500
#7  0x00000000009b8497 in execute_ex (ex=0x7fd9bbe14540) at /home/www/php-7.0.29/Zend/zend_vm_execute.h:414
#8  0x0000000000943cfa in zend_call_function (fci=0x7fffe3234bc0, fci_cache=0x7fffe3234b60)
    at /home/www/php-7.0.29/Zend/zend_execute_API.c:867
#9  0x00000000009831c8 in zend_call_method (object=0x0, obj_ce=0x0, fn_proxy=0x7fd9bbe7b960, 
    function_name=0x7fd9bbe7b8d8 "closure::__invoke\003", function_name_len=21, retval_ptr=0x0, param_count=1, 
    arg1=0x7fd9bbe14530, arg2=0x0) at /home/www/php-7.0.29/Zend/zend_interfaces.c:104
#10 0x0000000000771a4c in zif_spl_autoload_call (execute_data=0x7fd9bbe144d0, return_value=0x7fffe3234f70)
    at /home/www/php-7.0.29/ext/spl/php_spl.c:421
#11 0x0000000000943e12 in zend_call_function (fci=0x7fffe3234f20, fci_cache=0x7fffe3234ef0)
    at /home/www/php-7.0.29/Zend/zend_execute_API.c:887
#12 0x0000000000944508 in zend_lookup_class_ex (name=0x7fd9bbfe9f60, key=0x0, use_autoload=1)
    at /home/www/php-7.0.29/Zend/zend_execute_API.c:1049
#13 0x0000000000944e8c in zend_fetch_class (class_name=0x7fd9bbfe9f60, fetch_type=0)
    at /home/www/php-7.0.29/Zend/zend_execute_API.c:1373
#14 0x00000000009401d8 in zend_get_constant_ex (cname=0x7fd9bbfebc00, scope=0x7fd9b4aaa400, flags=0)
    at /home/www/php-7.0.29/Zend/zend_constants.c:351
#15 0x0000000000942bd6 in zval_update_constant_ex (p=0x7fd9bbfe8a18, inline_change=1 '\001', scope=0x7fd9b4aaa400)
    at /home/www/php-7.0.29/Zend/zend_execute_API.c:575
#16 0x00000000009621b7 in zend_update_class_constants (class_type=0x7fd9b4aaa400)
    at /home/www/php-7.0.29/Zend/zend_API.c:1134
#17 0x0000000000961d87 in zend_update_class_constants (class_type=0x7fd9b4aaa208)
    at /home/www/php-7.0.29/Zend/zend_API.c:1091
#18 0x0000000000962a5c in _object_and_properties_init (arg=0x7fd9bbfd3f38, class_type=0x7fd9b4aaa208, 
    properties=0x0, __zend_filename=0xf73538 "/home/www/php-7.0.29/ext/standard/var_unserializer.c", 
    __zend_lineno=467) at /home/www/php-7.0.29/Zend/zend_API.c:1291
#19 0x0000000000962b64 in _object_init_ex (arg=0x7fd9bbfd3f38, class_type=0x7fd9b4aaa208, 
    __zend_filename=0xf73538 "/home/www/php-7.0.29/ext/standard/var_unserializer.c", __zend_lineno=467)
    at /home/www/php-7.0.29/Zend/zend_API.c:1314
#20 0x000000000082d44d in object_common1 (rval=0x7fd9bbfd3f38, p=0x7fffe3235be0, max=0x7fffe323620e "", 
    var_hash=0x7fffe3235bd8, classes=0x0, ce=0x7fd9b4aaa208)
    at /home/www/php-7.0.29/ext/standard/var_unserializer.c:467
#21 0x000000000082f4b9 in php_var_unserialize_internal (rval=0x7fd9bbfd3f38, p=0x7fffe3235be0, 
    max=0x7fffe323620e "", var_hash=0x7fffe3235bd8, classes=0x0)
    at /home/www/php-7.0.29/ext/standard/var_unserializer.c:1231
#22 0x000000000082d139 in process_nested_data (rval=0x7fd9bbfcd510, p=0x7fffe3235be0, max=0x7fffe323620e "", 
---Type <return> to continue, or q <return> to quit---
    var_hash=0x7fffe3235bd8, classes=0x0, ht=0x7fd9bbfe9300, elements=5, objprops=1)
    at /home/www/php-7.0.29/ext/standard/var_unserializer.c:401
#23 0x000000000082d5a7 in object_common2 (rval=0x7fd9bbfcd510, p=0x7fffe3235be0, max=0x7fffe323620e "", 
    var_hash=0x7fffe3235bd8, classes=0x0, elements=7) at /home/www/php-7.0.29/ext/standard/var_unserializer.c:499
#24 0x000000000082f548 in php_var_unserialize_internal (rval=0x7fd9bbfcd510, p=0x7fffe3235be0, 
    max=0x7fffe323620e "", var_hash=0x7fffe3235bd8, classes=0x0)
    at /home/www/php-7.0.29/ext/standard/var_unserializer.c:1243
#25 0x000000000082d762 in php_var_unserialize_ex (rval=0x7fd9bbfcd510, p=0x7fffe3235be0, max=0x7fffe323620e "", 
    var_hash=0x7fffe3235bd8, classes=0x0) at /home/www/php-7.0.29/ext/standard/var_unserializer.c:533
#26 0x000000000082d6f9 in php_var_unserialize (rval=0x7fd9bbfcd510, p=0x7fffe3235be0, max=0x7fffe323620e "", 
    var_hash=0x7fffe3235bd8) at /home/www/php-7.0.29/ext/standard/var_unserializer.c:524
#27 0x00007fd9b55525ae in php_swoole_task_unpack (task_result=0x7fffe3235f98)
    at /home/www/swoole-2.1.3/swoole_server.c:405
#28 0x00007fd9b5554706 in php_swoole_onTask (serv=0x24fb130, req=0x7fffe3235f98)
    at /home/www/swoole-2.1.3/swoole_server.c:1026
#29 0x00007fd9b55f2f84 in swTaskWorker_onTask (pool=0x7fd9b4c5a230, task=0x7fffe3235f98)
    at /home/www/swoole-2.1.3/src/network/TaskWorker.c:69
#30 0x00007fd9b55f99b3 in swProcessPool_worker_loop (pool=0x7fd9b4c5a230, worker=0x7fd9b4c5ad88)
    at /home/www/swoole-2.1.3/src/network/ProcessPool.c:506
#31 0x00007fd9b55f933b in swProcessPool_spawn (worker=0x7fd9b4c5ad88)
    at /home/www/swoole-2.1.3/src/network/ProcessPool.c:368
#32 0x00007fd9b55f8997 in swProcessPool_start (pool=0x7fd9b4c5a230)
    at /home/www/swoole-2.1.3/src/network/ProcessPool.c:182
#33 0x00007fd9b5601922 in swManager_start (factory=0x24fb5c0) at /home/www/swoole-2.1.3/src/network/Manager.c:138
#34 0x00007fd9b55e44fd in swFactoryProcess_start (factory=0x24fb5c0)
    at /home/www/swoole-2.1.3/src/factory/FactoryProcess.c:86
#35 0x00007fd9b55eee8d in swServer_start (serv=0x24fb130) at /home/www/swoole-2.1.3/src/network/Server.c:755
#36 0x00007fd9b555aa20 in zim_swoole_server_start (execute_data=0x7fd9bbe14470, return_value=0x7fd9bbe14460)
    at /home/www/swoole-2.1.3/swoole_server.c:2689
#37 0x00000000009b9615 in ZEND_DO_FCALL_SPEC_HANDLER () at /home/www/php-7.0.29/Zend/zend_vm_execute.h:842
#38 0x00000000009b8497 in execute_ex (ex=0x7fd9bbe14030) at /home/www/php-7.0.29/Zend/zend_vm_execute.h:414
#39 0x00000000009b85a9 in zend_execute (op_array=0x7fd9bbe81000, return_value=0x0)
    at /home/www/php-7.0.29/Zend/zend_vm_execute.h:458
#40 0x000000000095ceaf in zend_execute_scripts (type=8, retval=0x0, file_count=3)
    at /home/www/php-7.0.29/Zend/zend.c:1445
#41 0x00000000008ce5c8 in php_execute_script (primary_file=0x7fffe323b740)
    at /home/www/php-7.0.29/main/main.c:2516
#42 0x0000000000a1d423 in do_cli (argc=2, argv=0x244aaf0) at /home/www/php-7.0.29/sapi/cli/php_cli.c:977
#43 0x0000000000a1e3dd in main (argc=2, argv=0x244aaf0) at /home/www/php-7.0.29/sapi/cli/php_cli.c:1347
```

挑一些重点的来说（前面的序号代表上面gdb的bt过程的序号，从后面往前分析）：

```txt
#29,#28：执行onTask回调，这还没到自定义的onTask回调（PHP回调），swoole会执行一些其他的代码，比如反序列化，然后才执行PHP回调。

#27-#24：把worker发过来的数据反序列化

#23-#14：把对象反序列化（初始化类的属性，然后初始化类的常量）

#13-#6：上面类的常量初始化需要加载其他类，所以这几步就是使用autload机制加载依赖的类，并找到相应文件包含进来。

#5-#0：编译文件（编译文件类、其他全局语句等）
```

上面的代码执行类似于下面类的反序列化（这也是后面的复现代码）：
```php
class MyRequest
{
    const PB = Protocol::PB;
    const JSON = Protocol::JSON;
}
```
反序列化MyRequest，初始化MyRequest的类属性（这里没有），然后初始化常量，发现常量依赖于别的类Protocol，所以又通过autoload加载类Protocol，然后编译Protocol类就报错了。

查看类的编译过程zend_compile_class_decl，编译类语句时会把当前类实体赋给`CG(active_class_entry)`，然后编译语句结束后恢复为原来的：

```c
void zend_compile_class_decl(zend_ast *ast) /* {{{ */
{
    // ...
    
    zend_class_entry *original_ce = CG(active_class_entry);
    znode original_implementing_class = FC(implementing_class);

    if (EXPECTED((decl->flags & ZEND_ACC_ANON_CLASS) == 0)) {
        zend_string *unqualified_name = decl->name;

        if (CG(active_class_entry)) {
            zend_error_noreturn(E_COMPILE_ERROR, "Class declarations may not be nested");   // 报错点
        }
    } else {
        // ...
    }
    
    // ...
    
    CG(active_class_entry) = ce;

    zend_compile_stmt(stmt_ast);    // 编译类语句

    // ...

    FC(implementing_class) = original_implementing_class;
    CG(active_class_entry) = original_ce;   // 恢复为原来的
}
/* }}} */
```

有人就会问了，为什么还要恢复原来的不是说不能嵌套定义吗？仔细看上面的代码，你会发现：出现那个错误走的if分支，还有else分支，else分支就是匿名类，也就是说PHP允许类里面定义匿名类的。

回到我们的问题，既然在编译类后恢复为原来的实体（我们的代码没有匿名类，所以original_ce为NULL），那么怎么会出现后面`CG(active_class_entry)`不为NULL呢？问题点出现在更新常量（上面的#17-#16）时，然后我们去看一下#17-#16的zend_update_class_constants函数（在php-7.0.29/Zend/zend_API.c）：
```C
ZEND_API int zend_update_class_constants(zend_class_entry *class_type) /* {{{ */
{
    if (!(class_type->ce_flags & ZEND_ACC_CONSTANTS_UPDATED)) {
        // ...
        if (!CE_STATIC_MEMBERS(class_type) && class_type->default_static_members_count) {
            // ...
        } else {
            zend_class_entry **scope = EG(current_execute_data) ? &EG(scope) : &CG(active_class_entry);
            zend_class_entry *old_scope = *scope;
            // ...
            *scope = class_type;
            // ...
        }
        class_type->ce_flags |= ZEND_ACC_CONSTANTS_UPDATED;
    }
    return SUCCESS;
}
/* }}} */
```
根据bt调用栈可以看出应该在1134行之前，所以上面的代码重点看这两句：
```
zend_class_entry **scope = EG(current_execute_data) ? &EG(scope) : &CG(active_class_entry);
*scope = class_type;
```
说明这里可以修改`CG(active_class_entry)`，执行上面第一句的时候`EG(current_execute_data)`如果不为NULL，就会把`CG(active_class_entry)`赋给scope，而后面那句scope修改也会影响`CG(active_class_entry)`。下面gdb调试：
```shell
(gdb) b php_swoole_onTask
Breakpoint 1 at 0x7fd9b5554657: file /home/www/swoole-2.1.3/swoole_server.c, line 1008.
(gdb) c
Continuing.

Breakpoint 1, php_swoole_onTask (serv=0x24fb130, req=0x7fffe3235f08)
    at /home/www/swoole-2.1.3/swoole_server.c:1008
1008        zval *zserv = (zval *) serv->ptr2;
(gdb) b zend_update_class_constants
Breakpoint 2 at 0x961d4d: file /home/www/php-7.0.29/Zend/zend_API.c, line 1089.
(gdb) c
Continuing.

Breakpoint 2, zend_update_class_constants (class_type=0x7fd9b4aaa2d0) at /home/www/php-7.0.29/Zend/zend_API.c:1089
1089        if (!(class_type->ce_flags & ZEND_ACC_CONSTANTS_UPDATED)) {
(gdb) until 1124

Breakpoint 2, zend_update_class_constants (class_type=0x7fd9b4aaa4c8) at /home/www/php-7.0.29/Zend/zend_API.c:1089
1089        if (!(class_type->ce_flags & ZEND_ACC_CONSTANTS_UPDATED)) {
(gdb) until 1124
zend_update_class_constants (class_type=0x7fd9b4aaa4c8) at /home/www/php-7.0.29/Zend/zend_API.c:1124
1124                zend_class_entry **scope = EG(current_execute_data) ? &EG(scope) : &CG(active_class_entry);
(gdb) display compiler_globals.active_class_entry 
1: compiler_globals.active_class_entry = (zend_class_entry *) 0x0
(gdb) n
1125                zend_class_entry *old_scope = *scope;
1: compiler_globals.active_class_entry = (zend_class_entry *) 0x0
(gdb) n
1130                *scope = class_type;
1: compiler_globals.active_class_entry = (zend_class_entry *) 0x0
(gdb) n
1131                ZEND_HASH_FOREACH_VAL(&class_type->constants_table, val) {
1: compiler_globals.active_class_entry = (zend_class_entry *) 0x7fd9b4aaa4c8
(gdb) p ((zend_class_entry *) 0x7fd9b4aaa4c8).name.val@30
$1 = // 业务代码，不方便显示
(gdb) c
Continuing.
warning: Temporarily disabling breakpoints for unloaded shared library "/home/www/bin/php-7.0.29-debug/lib/php/extensions/debug-non-zts-20151012/swoole.so"
[Inferior 1 (process 3875) exited with code 0377]
```
可以看出确实是1130行的`*scope = class_type`修改了`CG(active_class_entry)`的值。那么再来看看`zend_class_entry **scope = EG(current_execute_data) ? &EG(scope) : &CG(active_class_entry)`这句，为啥`EG(current_execute_data)`为NULL了呢？

先来看看这个`EG(current_execute_data)`，`EG(current_execute_data)`展开来就是executor_globals.current_execute_data，这是执行时的上下文信息，表示当前执行的是哪个execute_data。execute_data是zend_execute_data结构，是执行过程中最核心的一个结构，每次函数的调用、include/require、eval等都会生成一个新的结构，它表示当前的作用域、代码的执行位置以及局部变量的分配等等，等同于机器码执行过程中stack的角色（摘自[PHP7内核剖析](https://github.com/pangudashu/php7-internal/blob/master/3/zend_executor.md)）。`EG(current_execute_data)`为NULL，这是不可能，至少swoole启动时就调用了`swoole_server::start()`方法。

那到底是哪一步导致了`EG(current_execute_data)`为NULL？会不会是task worker启动时`EG(current_execute_data)`就为NULL了？

关闭服务，用gdb重新启动，开启多进程调试模式（因为用的是SWOOLE_PROCESS，test.php为入口文件）：

```shell
(gdb) set args test.php 
(gdb) set follow-fork-mode child 
(gdb) set detach-on-fork off
(gdb) b swProcessPool_spawn
Function "swProcessPool_spawn" not defined.
Make breakpoint pending on future shared library load? (y or [n]) y
Breakpoint 1 (swProcessPool_spawn) pending.
(gdb) r
Starting program: /usr/bin/php7.0.29-debug test.php 
// ...
[Switching to Thread 0x7ffff7fe1840 (LWP 4000)]

Breakpoint 1, swProcessPool_spawn (worker=0x7fffecc5ad88) at /home/www/swoole-2.1.3/src/network/ProcessPool.c:348
348        pid_t pid = fork();
(gdb) p executor_globals.current_execute_data 
$1 = (struct _zend_execute_data *) 0x7ffff3e14470
(gdb) b /home/www/swoole-2.1.3/src/network/ProcessPool.c:359
Breakpoint 2 at 0x7fffed5f92ed: /home/www/swoole-2.1.3/src/network/ProcessPool.c:359. (3 locations)
(gdb) c
Continuing.
[New process 4001]
// ...
[Switching to Thread 0x7ffff7fe1840 (LWP 4001)]

Breakpoint 2, swProcessPool_spawn (worker=0x7fffecc5ad88) at /home/www/swoole-2.1.3/src/network/ProcessPool.c:359
359            if (pool->onWorkerStart != NULL)
(gdb) display executor_globals.current_execute_data 
1: executor_globals.current_execute_data = (struct _zend_execute_data *) 0x7ffff3e14470
(gdb) n
361                pool->onWorkerStart(pool, worker->id);
1: executor_globals.current_execute_data = (struct _zend_execute_data *) 0x7ffff3e14470
(gdb) n
366            if (pool->main_loop)
1: executor_globals.current_execute_data = (struct _zend_execute_data *) 0x0
(gdb) c
Continuing.
```

从上面的分析可以看出：task worker初始化`EG(current_execute_data)`就为NULL了，但是一开始是不为NULL的，在执行完`pool->onWorkerStart(pool, worker->id)`后就变为NULL了，然后后面就进入worker的main_loop，也就是收worker的消息并处理。

再继续调试`pool->onWorkerStart(pool, worker->id)`里面的语句：
```
Breakpoint 2, swProcessPool_spawn (worker=0x7fffecc5ad88) at /home/www/swoole-2.1.3/src/network/ProcessPool.c:359
359            if (pool->onWorkerStart != NULL)
(gdb) s
361                pool->onWorkerStart(pool, worker->id);
(gdb) display executor_globals.current_execute_data
1: executor_globals.current_execute_data = (struct _zend_execute_data *) 0x7ffff3e14470
(gdb) s
swTaskWorker_onStart (pool=0x7fffecc5a230, worker_id=3) at /home/www/swoole-2.1.3/src/network/TaskWorker.c:121
121        swServer *serv = pool->ptr;
1: executor_globals.current_execute_data = (struct _zend_execute_data *) 0x7ffff3e14470
// ...
(gdb) n
130        swTaskWorker_signal_init();
1: executor_globals.current_execute_data = (struct _zend_execute_data *) 0x7ffff3e14470
(gdb) n
131        swWorker_onStart(serv);
1: executor_globals.current_execute_data = (struct _zend_execute_data *) 0x7ffff3e14470
(gdb) n
133        SwooleG.main_reactor = NULL;
1: executor_globals.current_execute_data = (struct _zend_execute_data *) 0x0
```
可以发现，在`swWorker_onStart(serv)`后就变为NULL了，看看函数体，出现问题只能是下面的这两句：
```C
void swWorker_onStart(swServer *serv)
{
    // ...
    if (serv->onWorkerStart)
    {
        serv->onWorkerStart(serv, SwooleWG.id); // serv->onWorkerStart = php_swoole_onWorkerStart
    }
}
```
再看`serv->onWorkerStart(serv, SwooleWG.id)`也就是`php_swoole_onWorkerStart(serv, SwooleWG.id)`：
```C
static void php_swoole_onWorkerStart(swServer *serv, int worker_id)
{
    // ...
#ifndef SW_COROUTINE
    zval **args[2];
    args[0] = &zserv;
    args[1] = &zworker_id;
    if (sw_call_user_function_ex(EG(function_table), NULL, php_sw_server_callbacks[SW_SERVER_CB_onWorkerStart], &retval, 2, args, 0, NULL TSRMLS_CC) == FAILURE)
    {
        swoole_php_fatal_error(E_WARNING, "onWorkerStart handler error.");
    }
#else
    zval *args[2];
    args[0] = zserv;
    args[1] = zworker_id;

    zend_fcall_info_cache *cache = php_sw_server_caches[SW_SERVER_CB_onWorkerStart];
    int ret = coro_create(cache, args, 2, &retval, NULL, NULL);
    if (ret != 0)
    {
        sw_zval_ptr_dtor(&zworker_id);
        if (ret == CORO_LIMIT)
        {
            swWarn("Failed to handle onWorkerStart. Coroutine limited.");
        }
        return;
    }
#endif
    // ...
}
```

到这里基本可以确定就是协程的问题了，没有定义SW_COROUTINE宏就调用PHP内核提供的call_user_function函数API执行`php_sw_server_callbacks[SW_SERVER_CB_onWorkerStart]`（也就是用户自定义的onWorkerStart回调），有定义SW_COROUTINE宏就调用使用swoole自定义`coro_create`创建协程，使用协程去调用`php_sw_server_callbacks[SW_SERVER_CB_onWorkerStart]`。swoole 2.1.3默认启用协程（不启用编译都通不过，这个你们可以去试试），所以这里会调用后者去执行用户自定义的回调，4.0后改为了[可配置的协程方式](https://wiki.swoole.com/wiki/page/949.html)。

再看看为啥这样？来看`sw_coro_create()`函数：
```C
int sw_coro_create(zend_fcall_info_cache *fci_cache, zval **argv, int argc, zval *retval, void *post_callback, void* params)
{
    // ...
    call->symbol_table = NULL;
    SW_ALLOC_INIT_ZVAL(retval);
    COROG.allocated_return_value_ptr = retval;
    EG(current_execute_data) = NULL;
    zend_init_execute_data(call, op_array, retval);

    // ...
    
    int coro_status;
    if (!setjmp(*swReactorCheckPoint))
    {
        zend_execute_ex(call);
        coro_close(TSRMLS_C);
        swTraceLog(SW_TRACE_COROUTINE, "[CORO_END] Create the %d coro with stack. heap size: %zu", COROG.coro_num, zend_memory_usage(0));
        coro_status = CORO_END;
    }
    else
    {
        coro_status = CORO_YIELD;
    }
    
    COROG.require = 0;
    return coro_status;
}
```

`EG(current_execute_data) = NULL`问题就出在这里，把值修改为NULL了。我觉得应该保存上下文，在协程执行完`zend_execute_ex(call)`后恢复上下文。本人对swoole的协程实现还不是很了解，不知道他们的实现方式，所以暂时没法修复这个bug，这个bug在4.0后是好的，据swoole组的说因为4.0重构了协程的实现，所以不会出现这个问题，建议升级到最新版4.0.3。

# 0x03 小结
从上面的分析过程可以看出：是协程实现在`onWorkerStart`时把上下文信息`EG(current_execute_data)`“弄丢”了，导致后面反序列化时判断错误使用了一个类的实体(zend_class_entry)作为`CG(active_class_entry)`，然后解析另一个类，导致了错误`Class declarations may not be nested`。


这个问题已向swoole官方反馈，官方回应已收到，问题还在解决中。

# 0x04 附录一：复现步骤
文件列表：
```shell
$ ll
总用量 16
-rw-r--r-- 1 www www  122 8月   7 11:10 MyRequest.php
-rw-r--r-- 1 www www  339 8月   7 11:11 Protocol.php
-rw-r--r-- 1 www www  262 8月   7 11:19 swoole_client.php
-rw-r--r-- 1 www www 1466 8月   7 11:23 swoole_server.php
```
文件说明：
```
swoole_server.php：提供服务。
swoole_client.php：客户端。
MyRequest.php：被序列化的类MyRequest所在文件。
Protocol.php：类MyRequest依赖的类Protocol所在文件。
```
swoole_server.php：
```php
<?php
class Autoloader
{
    public static function load($class)
    {
        $ext = '.php';
        $class = str_replace('\\', '/', $class);
        $path = __DIR__ . DIRECTORY_SEPARATOR . $class . $ext;
        require $path;
    }
}

spl_autoload_register(['Autoloader', 'load']);

function my_log($content, $file = null)
{
    if ($file === null) {
        $file = __DIR__ . DIRECTORY_SEPARATOR . 'log.log';
    }
    $content = '[' . \date('Y-m-d H:i:s') . ']' . print_r($content, 1) . "\r\n";
    return @file_put_contents($file, $content, FILE_APPEND);
}


$server = new \swoole_server('127.0.0.1', 12222, \SWOOLE_PROCESS);
$server->set([
    'daemonize' => 1,
    'worker_num' => 2,
    'task_worker_num' => 4,
    'log_file' => __DIR__ . DIRECTORY_SEPARATOR . 'swoole.log'
]);
$server->on('receive', function(\swoole_server $server, $fd, $from_id, $data) {
    $request = new \MyRequest();
    $server->task($request); // 这里会序列化MyRequest
});

$server->on('task', function(\swoole_server $server, $task_id, $from_id, $taskData) {
    my_log("on task");
});

$server->on('finish', function(\swoole_server $server, $task_id, $from_id, $taskData) {
    my_log("on finish");
});

$server->on('start', function(\swoole_server $server) {
    my_log("on start");
});

$server->on('workerstart', function(\swoole_server $server, int $worker_id) {
    my_log("on workerstart");
});

$server->start();
```
swoole_client.php：

```PHP
<?php
$client = new \swoole_client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_SYNC);
$client->connect('127.0.0.1', 12222, 30);

/* 随便发的，为的是触发服务的onTask回调 */
$body = "test";
$packed = pack('N', strlen($body)) . $body;
$client->send($packed);
```

MyRequest.php：

```php
<?php
/**
 * Class MyRequest
 */
class MyRequest
{
    const PB = Protocol::PB;
    const JSON = Protocol::JSON;
}
```

Protocol.php：

```php
<?php
/**
 * Protocol enum
 */
final class Protocol
{
    const PB = 1;
    const JSON = 2;

    public function getEnumValues()
    {
        return array(
            'PB' => self::PB,
            'JSON' => self::JSON,
        );
    }
}
```

复现步骤：

```shell
[www@host-XXX swoole_unserialize]$ pwd
/home/www/swoole_unserialize
[www@host-XXX swoole_unserialize]$ php7.0.29-debug swoole_server.php 
[www@host-XXX swoole_unserialize]$ ps -ef|grep swoole_server.php
www       4785     1  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4786  4785  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4788  4786  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4789  4786  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4790  4786  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4791  4786  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4792  4786  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4793  4786  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4795  9545  0 11:40 pts/3    00:00:00 grep --color=auto swoole_server.php
[www@host-XXX swoole_unserialize]$ php7.0.29-debug swoole_client.php 
[www@host-XXX swoole_unserialize]$ ps -ef|grep swoole
www       4785     1  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4786  4785  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4789  4786  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4790  4786  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4791  4786  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4792  4786  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4793  4786  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4797  4786  0 11:40 ?        00:00:00 php7.0.29-debug swoole_server.php
www       4799  9545  0 11:40 pts/3    00:00:00 grep --color=auto swoole
[www@host-XXX swoole_unserialize]$ tail -2 swoole.log 
[2018-08-07 11:40:40 ^4788.2]	ERROR	zm_deactivate_swoole (ERROR 503): Fatal error: Class declarations may not be nested in /home/www/swoole_unserialize/Protocol.php on line 5.
[2018-08-07 11:40:40 $4786.0]	WARNING	swManager_check_exit_status: worker#2 abnormal exit, status=255, signal=0
```

两次`ps -ef|grep swoole`可以看出进程ID为4788的进程（task worker进程）在请求完就挂掉了，然后manager进程(进程ID：4786)会重新拉起一个task worker进程，ID为4797。因为设置了log_file，所以可以查看swoole.log。`tail -2 swoole.log`第一条的`^4788`表示进程ID为4788的task worker的日志，错误信息为`Fatal error: Class declarations may not be nested...`。`$4786`表示Manager进程的信息，Manager收到子进程4788退出的信号，于是打印信息，`abnormal exit, status=255`表示异常退出。