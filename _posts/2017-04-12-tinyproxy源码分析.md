---
layout: post
title: tinyproxy源码分析
date: 2017-04-12 20:17:19 +0800
comments: false
---

### 重要的全局变量

```
static vector_t listen_fds;// listen端口fd

static struct child_s *child_ptr; // 子进程信息数组

static unsigned int *servers_waiting;  // 等待连接的子进程，也就是空闲状态

unsigned int received_sighup = FALSE;
```
### main函数及注解

```
int
main (int argc, char **argv)
{
        // 设置文件权限
        umask (0177);

        log_message (LOG_INFO, "Initializing " PACKAGE " ...");

		// 解析配置文件
        if (config_compile_regex()) {
                exit (EX_SOFTWARE);
        }

		// 加载默认配置
        initialize_config_defaults (&config_defaults);

        // 解析命令行
        process_cmdline (argc, argv, &config_defaults);

		// 加载配置文件配置
        if (reload_config_file (config_defaults.config_file,
                                &config,
                                &config_defaults)) {
                exit (EX_SOFTWARE);
        }

		// 初始化profiling相关内存
        init_stats ();

        // 匿名代理处理
        if (is_anonymous_enabled ()) {
                anonymous_insert ("Content-Length");
                anonymous_insert ("Content-Type");
        }

		// 守护进程
        if (config.godaemon == TRUE)
                makedaemon ();

		// 注册SIGPIPE 防异常退出。对关闭连接的操作会产生这个信号
        if (set_signal_handler (SIGPIPE, SIG_IGN) == SIG_ERR) {
                fprintf (stderr, "%s: Could not set the \"SIGPIPE\" signal.\n",
                         argv[0]);
                exit (EX_OSERR);
        }

#ifdef FILTER_ENABLE
        if (config.filter)
                filter_init ();
#endif /* FILTER_ENABLE */

        // 获得监听端口的fd
        if (child_listening_sockets(config.listen_addrs, config.port) < 0) {
                fprintf (stderr, "%s: Could not create listening sockets.\n",
                         argv[0]);
                exit (EX_OSERR);
        }

        // 如果是root运行则切换用户
        if (geteuid () == 0)
                change_user (argv[0]);
        else
                log_message (LOG_WARNING,
                             "Not running as root, so not changing UID/GID.");

       // 创建日志
        if (setup_logging ()) {
                exit (EX_SOFTWARE);
        }

        /* Create pid file after we drop privileges */
        if (config.pidpath) {
                if (pidfile_create (config.pidpath) < 0) {
                        fprintf (stderr, "%s: Could not create PID file.\n",
                                 argv[0]);
                        exit (EX_OSERR);
                }
        }

		// 初始化子进程相关信息，并在此处逐个启动子进程，子进程数由配置中的startserver决定
        if (child_pool_create () < 0) {
                fprintf (stderr,
                         "%s: Could not create the pool of children.\n",
                         argv[0]);
                exit (EX_SOFTWARE);
        }

        /* These signals are only for the parent process. */
        log_message (LOG_INFO, "Setting the various signals.");

		// 注册SIGCHLD 在taskesig中waitpid非阻塞等待子进程退出
        if (set_signal_handler (SIGCHLD, takesig) == SIG_ERR) {
                fprintf (stderr, "%s: Could not set the \"SIGCHLD\" signal.\n",
                         argv[0]);
                exit (EX_OSERR);
        }

		//注册SIGTERM ，处理父进程退出事件
        if (set_signal_handler (SIGTERM, takesig) == SIG_ERR) {
                fprintf (stderr, "%s: Could not set the \"SIGTERM\" signal.\n",
                         argv[0]);
                exit (EX_OSERR);
        }


		// 收到SIGHUP杀死所有子进程
        if (set_signal_handler (SIGHUP, takesig) == SIG_ERR) {
                fprintf (stderr, "%s: Could not set the \"SIGHUP\" signal.\n",
                         argv[0]);
                exit (EX_OSERR);
        }

        /* Start the main loop */
        log_message (LOG_INFO, "Starting main loop. Accepting connections.");

		// 其实是父进程循环，当收到SIGTERM时退出
        child_main_loop ();

        log_message (LOG_INFO, "Shutting down.");

		// 向子进程发送退出信号
        child_kill_children (SIGTERM);

        // 关闭listenfds中的fd，并且释放相关内存
        child_close_sock ();

        /* Remove the PID file */
        if (unlink (config.pidpath) < 0) {
                log_message (LOG_WARNING,
                             "Could not remove PID file \"%s\": %s.",
                             config.pidpath, strerror (errno));
        }

#ifdef FILTER_ENABLE
        if (config.filter)
                filter_destroy ();
#endif /* FILTER_ENABLE */

        shutdown_logging ();

        return EXIT_SUCCESS;
}
```

可以看到，tinyproxy的main函数启动结构非常简单，是一个最基本的多进程unix服务端程序。其工作核心就在child.c/child_main函数中完成的

### 父进程的工作

父进程主要起到一个增加子进程，当等待的子进程小于了配置的最小空闲数后，则在最大进程数的范围内，新开子线程；另外也负责在收到关闭信号后，关闭所有子进程。而进程数收缩的工作，由子线程自身在每次完成一次转发后review当前空闲的连接是否大于了最大空闲数，检查后会结束自己的声明。

### 子进程的工作

子进程的实现就是一个个worker。单个worker使用select对父进程fork出来的listenfds进行监听，当有连接事件发生时，worker通过accept获取到客户端socket的资源，然后读取连接数据做代理转发。

handle_connection函数是主要的处理流程。顺序读取socket中的数据，以http协议的约定解析。如第一行数据是协议版本，之后每一行都是http header数据，直到空行。tinyproxy最多支持10000个header头，宏的形式写死在代码中。

#### 存储headers的方式
tinyproxy解析到一个请求，会将他的请求头放到一个哈希表中。哈希函数如下：

```
for (hash = seed; *key != '\0'; key++) {
                hash = ((hash << 5) + hash) ^ tolower (*key);
        }
```
seed是一个随机数，其中的数学原理，还没有参透，改日再细看有什么玄机。（= =）
这个哈希表默认是一个大小4096的拉链法哈希表。

存储请求中的请求头后，再将配置文件中定义的用户自定义请求头写入这个哈希表。

#### 检查请求头
此时一个请求的方法体还没有被从连接缓冲区中读出，因为还要对请求头做更多的检查。如解析http版本号，http method等；还比如支持https的重要对Connect方法的检查。在配置文件中必须指定接受Connect方法的端口。

```
...
 } else if (strcmp (request->method, "CONNECT") == 0) {
                if (extract_url (url, HTTP_PORT_SSL, request) < 0) {
                        indicate_http_error (connptr, 400, "Bad Request",
                                             "detail", "Could not parse URL",
                                             "url", url, NULL);
                        goto fail;
                }

                /* Verify that the port in the CONNECT method is allowed */
                if (!check_allowed_connect_ports (request->port,
                                                  config.connect_ports))
                {
                        indicate_http_error (connptr, 403, "Access violation",
                                             "detail",
                                             "The CONNECT method not allowed "
                                             "with the port you tried to use.",
                                             "url", url, NULL);
                        log_message (LOG_INFO,
                                     "Refused CONNECT method on port %d",
                                     request->port);
                        goto fail;
                }

                connptr->connect_method = TRUE;
        } else {
        ....
```

#### 完成代理转发
之后就很简单了，根据请求头中指定的目标地址，建立tcp连接，将哈希表中的http头写入连接，之后“一次性”读出客户连接中的所有数据，再向目标连接replay。这里的一次性考虑到buffer大小，其实有可能是分段写的。

### 总结

之前写过一段时间unix后端程序，业务框架很重，对unix编程没有什么太深的感受。本次学习tinyproxy也是为了顺便熟悉一下unix编程。再次感受到了unix/C简单高效的编程之美，比干啃书有意思很多（逃）
