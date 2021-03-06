#Nginx源码分析之启动过程

nginx的启动过程代码主要分布在src/core以及src/os/unix目录下。启动流程的函数调用序列：main(src/core/nginx.c)→ngx_init_cycle(src/core/ngx_cycle.c)→ngx_master_process_cycle(src/os/)。nginx的启动过程就是围绕着这三个函数进行的。

main函数的处理过程总体上可以概括如下：

1. 简单初始化一些数据接结构和模块：ngx_debug_init、ngx_strerror_init、ngx_time_init、ngx_regex_init、ngx_log_init、ngx_ssl_init等
2. 获取并处理命令参数：ngx_get_options。
3. 初始化ngx_cycle_t结构体变量init_cycle，设置其log、pool字段等。
	<pre>
	ngx_memzero(&init_cycle, sizeof(ngx_cycle_t));
    init_cycle.log = log;
    ngx_cycle = &init_cycle;

    init_cycle.pool = ngx_create_pool(1024, log);
    if (init_cycle.pool == NULL) {
        return 1;
    }
</pre>
4. 保存参数，设置全局变量：ngx_argc ngx_os_argv ngx_argv ngx_environ
5. 处理控制台命令行参数，设置init_cycle的字段。这些字段包括：conf_prefix、prefix（-p prefix）、conf_file(-c filename)、conf_param(-g directives)。此外，还设置init_cyle.log.log_level=NGX_LOG_INFO。
6. 调用ngx_os_init来设置一些和操作系统相关的全局变量：ngx_page_size、ngx_cacheline_size、ngx_ncpu、ngx_max_sockets、ngx_inherited_nonblocking。
7. 调用ngx_crc32_table_init初始化ngx_crc32_table_short。用于后续做crc校验。
8. 调用ngx_add_inherited_sockets(&init_cycle)→ngx_set_inherited_sockets，初始化init_cycle的listening字段（一个ngx_listening_t的数组）。
9. 对所有模块进行计数
	<pre>
	ngx_max_module = 0;
    for (i = 0; ngx_modules[i]; i++) {
        ngx_modules[i]->index = ngx_max_module++;
    }
</pre>
10. 调用ngx_init_cycle进行模块的初始化，当解析配置文件错误时，退出程序。这个函数传入init_cycle然后返回一个新的ngx_cycle_t。
11. 调用ngx_signal_process、ngx_init_signals处理信号。
12. 在daemon模式下，调用ngx_daemon以守护进程的方式运行。这里可以在./configure的时候加入参数--with-debug，并在nginx.conf中配置:
	<pre>
	master_process  off; # 简化调试 此指令不得用于生产环境
	daemon          off; # 简化调试 此指令可以用到生产环境
</pre>
可以取消守护进程模式以及master线程模型。
13. 调用ngx_create_pidfile创建pid文件，把master进程的pid保存在里面。
14. 根据进程模式来分别调用相应的函数
	<pre>
	if (ngx_process == NGX_PROCESS_SINGLE) {
        ngx_single_process_cycle(cycle);

    } else {
        ngx_master_process_cycle(cycle);
    }
</pre>
多进程的情况下，调用ngx_master_process_cycle。单进程的情况下调用ngx_single_process_cycle完成最后的启动工作。
	
整个启动过程中一个关键的变量init_cycle，其数据结构ngx_cycle_t如下所示：

<pre>
struct ngx_cycle_s {
    void                  ****conf_ctx;
    ngx_pool_t               *pool;

    ngx_log_t                *log;
    ngx_log_t                 new_log;

    ngx_connection_t        **files;
    ngx_connection_t         *free_connections;
    ngx_uint_t                free_connection_n;

    ngx_queue_t               reusable_connections_queue;

    ngx_array_t               listening;
    ngx_array_t               paths;
    ngx_list_t                open_files;
    ngx_list_t                shared_memory;

    ngx_uint_t                connection_n;
    ngx_uint_t                files_n;

    ngx_connection_t         *connections;
    ngx_event_t              *read_events;
    ngx_event_t              *write_events;

    ngx_cycle_t              *old_cycle;

    ngx_str_t                 conf_file;
    ngx_str_t                 conf_param;
    ngx_str_t                 conf_prefix;
    ngx_str_t                 prefix;
    ngx_str_t                 lock_file;
    ngx_str_t                 hostname;
};
</pre>

它保存了一次启动过程需要的一些资源。

ngx_init_cycle函数的处理过程如下：

1. 调用ngx_timezone_update()、ngx_timeofday、ngx_time_update()来更新时区、时间等，做时间校准，用来创建定时器等。
2. 创建pool,赋给一个新的cycle（ngx_cycle_t）。这个新的cycle的一些字段从旧的cycle传递过来，比如：log,conf_prefix,prefix,conf_file,conf_param。
	<pre>
	cycle->conf_prefix.len = old_cycle->conf_prefix.len;
    cycle->conf_prefix.data = ngx_pstrdup(pool, &old_cycle->conf_prefix);
    if (cycle->conf_prefix.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->prefix.len = old_cycle->prefix.len;
    cycle->prefix.data = ngx_pstrdup(pool, &old_cycle->prefix);
    if (cycle->prefix.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->conf_file.len = old_cycle->conf_file.len;
    cycle->conf_file.data = ngx_pnalloc(pool, old_cycle->conf_file.len + 1);
    if (cycle->conf_file.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }
    ngx_cpystrn(cycle->conf_file.data, old_cycle->conf_file.data,
                old_cycle->conf_file.len + 1);

    cycle->conf_param.len = old_cycle->conf_param.len;
    cycle->conf_param.data = ngx_pstrdup(pool, &old_cycle->conf_param);
    if (cycle->conf_param.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }
	</pre>
还有一些字段会首先判断old_cycle中是否存在，如果存在，则申请同样大小的空间，并初始化。这些字段如下：
	<pre>
	n = old_cycle->paths.nelts ? old_cycle->paths.nelts : 10;

	//paths
    cycle->paths.elts = ngx_pcalloc(pool, n * sizeof(ngx_path_t *));
    if (cycle->paths.elts == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->paths.nelts = 0;
    cycle->paths.size = sizeof(ngx_path_t *);
    cycle->paths.nalloc = n;
    cycle->paths.pool = pool;


    //open_files
    if (old_cycle->open_files.part.nelts) {
        n = old_cycle->open_files.part.nelts;
        for (part = old_cycle->open_files.part.next; part; part = part->next) {
            n += part->nelts;
        }

    } else {
        n = 20;
    }
    
    //shared_memory
    if (old_cycle->shared_memory.part.nelts) {
        n = old_cycle->shared_memory.part.nelts;
        for (part = old_cycle->shared_memory.part.next; part; part = part->next)
        {
            n += part->nelts;
        }

    } else {
        n = 1;
    }
    
    //listening
    n = old_cycle->listening.nelts ? old_cycle->listening.nelts : 10;

    cycle->listening.elts = ngx_pcalloc(pool, n * sizeof(ngx_listening_t));
    if (cycle->listening.elts == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->listening.nelts = 0;
    cycle->listening.size = sizeof(ngx_listening_t);
    cycle->listening.nalloc = n;
    cycle->listening.pool = pool;
</pre>
	
	此外，new_log.log_level重新赋值的为NGX_LOG_ERR；old_cycle为传递进来的cycle；hostname为gethostname;初始化resuable_connection_queue。

	这里有一个关键变量的初始化：conf_ctx。初始化为ngx_max_module个void *指针。说明其实所有模块的配置结构的指针。

3. 调用所有模块的create_conf，返回的配置结构指针放到conf_ctx数组中，索引为ngx_modules[i]->index。
4. 从命令行和配置文件读取配置更新到conf_ctx中。ngx_conf_param是读取命令行中的指令，ngx_conf_parse是把读取配置文件。ngx_conf_param最后也是通过调用ngx_cong_parse来读取配置的。ngx_conf_parse函数中有一个for循环，每次都调用ngx_conf_read_token取得一个配置指令，然后调用ngx_conf_handler来处理这条指令。ngx_conf_handler每次会遍历所有模块的指令集，查找这条配置指令并分析其合法性，如果正确则创建配置结构并把指针加入到cycle.conf_ctx中。
	遍历指令集的过程首先是遍历所有的核心类模块，若是 event类的指令，则会遍历到ngx_events_module，这个模块是属于核心类的，其钩子set又会嵌套调用ngx_conf_parse去遍历所有的event类模块，同样的，若是http类指令，则会遍历到ngx_http_module，该模块的钩子set进一步遍历所有的http类模块，mail类指令会遍历到ngx_mail_module，该模块的钩子进一步遍历到所有的mail类模块。要特别注意的是：这三个遍历过程中会在适当的时机调用event类模块、http类模块和mail类模块的创建配置和初始化配置的钩子。从这里可以看出，event、http、mail三类模块的钩子是配置中的指令驱动的。
5. 调用core module的init_conf。
6. 读取核心模块ngx_core_module的配置结构，调用ngx_create_pidfile创建pid文件。
	<pre>
	ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
</pre>
	
	这里代码中有一句注释：
	<pre>
     we do not create the pid file in the first ngx_init_cycle() call
     because we need to write the demonized process pid
</pre>
	当不是第一次初始化cycles时才会调用ngx_create_pidfile写入pid。
7. 调用ngx_test_lockfile,ngx_create_paths并打开error_log文件复制给cycle->new_log.file。
8. 遍历cycle的open_files.part.elts，打开每一个文件。open_files填充的文件数据是读取配置文件时写入的。
9. 创建共享内存。这里和对open_files类似。先预分配空间，再填充数据。
10. 处理listening sockets，遍历cycle->listening数组与old_cycle->listenning进行比较，设置cycle->listening的一些状态信息，调用ngx_open_listening_sockets启动所有监听socket，循环调用socket、bind、listen完成服务端监听监听socket的启动。并调用ngx_configure_listening_sockets配置监听socket,根据ngx_listening_t中的状态信息设置socket的读写缓存和TCP_DEFER_ACCEPT。
11. 调用每个module的init_module。
12. 关闭或者删除一些残留在old_cycle中的资源，首先释放不用的共性内存，关闭不使用的监听socket，再关闭不适用的打开文件。最后把old_cycle放入ngx_old_cycles。最后再设定一个定时器，定期回调ngx_cleaner_event清理ngx_old_cycles。周期设置为30000ms。


接下来是进程的启动，包括master和worker进程。main函数最后调用ngx_master_process_cycle来启动master进程模式(这里对单进程模式不做讲述)。

1. 设置一些信号，如下：
	<pre>
	sigaddset(&set, SIGCHLD);
    sigaddset(&set, SIGALRM);
    sigaddset(&set, SIGIO);
    sigaddset(&set, SIGINT);
    sigaddset(&set, ngx_signal_value(NGX_RECONFIGURE_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_REOPEN_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_NOACCEPT_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_TERMINATE_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_CHANGEBIN_SIGNAL));
</pre> 
2. 调用ngx_setproctitle设置进程标题："master process" + ngx_argv[0...]
3. 启动worker进程,数量为ccf->worker_processes。
	<pre>
	ngx_start_worker_processes(cycle, ccf->worker_processes,
                               NGX_PROCESS_RESPAWN);
</pre>
4. 启动文件cache管理进程。
	<pre>
	ngx_start_cache_manager_processes(cycle, 0);
</pre>
	这里的cahche在一些模块中是需要的，如fastcgi模块等,这些模块会把文件cache路径添加到cycle->paths中，文件cache管理进程会定期调用这些模块的文件cache处理钩子处理一下文件cache。
5. master主循环，主要是循环处理信号量。在循环过程中，判断相应的条件然后进入相应的处理。这里的相关标志位基本都是在信号处理函数中赋值的。
	<pre>
	for ( ;; ) {		// delay用来设置等待worker退出的时间，master接收了退出信号后首先发送退出信号给worker，		// 而worker退出需要一些时间		if (delay) {			delay *= 2;				...			itv.it_interval.tv_sec = 0;			itv.it_interval.tv_usec = 0;			itv.it_value.tv_sec = delay / 1000;			itv.it_value.tv_usec = (delay % 1000 ) * 1000;			// 设置定时器			if (setitimer(ITIMER_REAL, &itv, NULL) == -1) {				ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,				“setitimer() failed”);			}		}		ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, “sigsuspend”);		// 挂起信号量，等待定时器		sigsuspend(&set);		ngx_time_update(0, 0);		ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, “wake up”);		// 收到了SIGCHLD信号，有worker退出（ngx_reap==1）		if (ngx_reap) {			ngx_reap = 0;			ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, “reap children”);			// 处理所有worker，如果有worker异常退出则重启这个worker，如果所有worker都退出			// 返回0赋值给live			live = ngx_reap_children(cycle);		}		// 如果worker都已经退出，		// 并且收到了NGX_CMD_TERMINATE命令或者SIGTERM信号或者SIGINT信号(ngx_terminate=1)		// 或者NGX_CMD_QUIT命令或者SIGQUIT信号(ngx_quit=1)，则master退出		if (!live && (ngx_terminate || ngx_quit)) {			ngx_master_process_exit(cycle);		}		// 收到了NGX_CMD_TERMINATE命令或者SIGTERM信号或者SIGINT信号，		// 通知所有worker退出，并且等待worker退出		if (ngx_terminate) {			// 设置延时			if (delay == 0) {				delay = 50;			}			if (delay > 1000) {				// 延时已到，给所有worker发送SIGKILL信号，强制杀死worker				ngx_signal_worker_processes(cycle, SIGKILL);			} else {				// 给所有worker发送SIGTERM信号，通知worker退出				ngx_signal_worker_processes(cycle,				ngx_signal_value(NGX_TERMINATE_SIGNAL));			}			continue;		}		// 收到了NGX_CMD_QUIT命令或者SIGQUIT信号		if (ngx_quit) {			// 给所有worker发送SIGQUIT信号			ngx_signal_worker_processes(cycle,			ngx_signal_value(NGX_SHUTDOWN_SIGNAL));			// 关闭所有监听的socket			ls = cycle->listening.elts;			for (n = 0; n < cycle->listening.nelts; n++) {				if (ngx_close_socket(ls[n].fd) == -1) {					ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_socket_errno,						ngx_close_socket_n ” %V failed”,&ls[n].addr_text);				}			}			cycle->listening.nelts = 0;		continue;		}		// 收到了SIGHUP信号		if (ngx_reconfigure) {			ngx_reconfigure = 0;			// 代码已经被替换，重启worker，不需要重新初始化配置			if (ngx_new_binary) {				ngx_start_worker_processes(cycle, ccf->worker_processes,					NGX_PROCESS_RESPAWN);				ngx_start_cache_manager_processes(cycle, 0);				ngx_noaccepting = 0;				continue;			}			ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, “reconfiguring”);			// 重新初始化配置			cycle = ngx_init_cycle(cycle);			if (cycle == NULL) {				cycle = (ngx_cycle_t *) ngx_cycle;				continue;			}			// 重启worker			ngx_cycle = cycle;			ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx,			ngx_core_module);			ngx_start_worker_processes(cycle, ccf->worker_processes,				NGX_PROCESS_JUST_RESPAWN);			ngx_start_cache_manager_processes(cycle, 1);			live = 1;			ngx_signal_worker_processes(cycle,				ngx_signal_value(NGX_SHUTDOWN_SIGNAL));		}		// 当ngx_noaccepting=1的时候会把ngx_restart设为1，重启worker		if (ngx_restart) {			ngx_restart = 0;			ngx_start_worker_processes(cycle, ccf->worker_processes,				NGX_PROCESS_RESPAWN);			ngx_start_cache_manager_processes(cycle, 0);			live = 1;		}		// 收到SIGUSR1信号，重新打开log文件		if (ngx_reopen) {			ngx_reopen = 0;			ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, “reopening logs”);			ngx_reopen_files(cycle, ccf->user);			ngx_signal_worker_processes(cycle,			ngx_signal_value(NGX_REOPEN_SIGNAL));		}		// 收到SIGUSR2信号，热代码替换		if (ngx_change_binary) {			ngx_change_binary = 0;			ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, “changing binary”);			// 调用execve执行新的代码			ngx_new_binary = ngx_exec_new_binary(cycle, ngx_argv);		}		// 收到SIGWINCH信号，不再接收请求，worker退出，master不退出		if (ngx_noaccept) {			ngx_noaccept = 0;			ngx_noaccepting = 1;			ngx_signal_worker_processes(cycle,			ngx_signal_value(NGX_SHUTDOWN_SIGNAL));		}	}
</pre>
信号处理函数是在main函数中进行的初始化：ngx_init_signals(cycle->log)。其中的signal handler代码如下所示：
<pre>
void
ngx_signal_handler(int signo)
{
    char            *action;
    ngx_int_t        ignore;
    ngx_err_t        err;
    ngx_signal_t    *sig;

    ignore = 0;

    err = ngx_errno;

    for (sig = signals; sig->signo != 0; sig++) {
        if (sig->signo == signo) {
            break;
        }
    }

    ngx_time_sigsafe_update();

    action = "";

    switch (ngx_process) {

    case NGX_PROCESS_MASTER:
    case NGX_PROCESS_SINGLE:
        switch (signo) {

        case ngx_signal_value(NGX_SHUTDOWN_SIGNAL):
            ngx_quit = 1;
            action = ", shutting down";
            break;

        case ngx_signal_value(NGX_TERMINATE_SIGNAL):
        case SIGINT:
            ngx_terminate = 1;
            action = ", exiting";
            break;

        case ngx_signal_value(NGX_NOACCEPT_SIGNAL):
            if (ngx_daemonized) {
                ngx_noaccept = 1;
                action = ", stop accepting connections";
            }
            break;

        case ngx_signal_value(NGX_RECONFIGURE_SIGNAL):
            ngx_reconfigure = 1;
            action = ", reconfiguring";
            break;

        case ngx_signal_value(NGX_REOPEN_SIGNAL):
            ngx_reopen = 1;
            action = ", reopening logs";
            break;

        case ngx_signal_value(NGX_CHANGEBIN_SIGNAL):
            if (getppid() > 1 || ngx_new_binary > 0) {

                /*
                 * Ignore the signal in the new binary if its parent is
                 * not the init process, i.e. the old binary's process
                 * is still running.  Or ignore the signal in the old binary's
                 * process if the new binary's process is already running.
                 */

                action = ", ignoring";
                ignore = 1;
                break;
            }

            ngx_change_binary = 1;
            action = ", changing binary";
            break;

        case SIGALRM:
            ngx_sigalrm = 1;
            break;

        case SIGIO:
            ngx_sigio = 1;
            break;

        case SIGCHLD:
            ngx_reap = 1;
            break;
        }

        break;

    case NGX_PROCESS_WORKER:
    case NGX_PROCESS_HELPER:
        switch (signo) {

        case ngx_signal_value(NGX_NOACCEPT_SIGNAL):
            if (!ngx_daemonized) {
                break;
            }
            ngx_debug_quit = 1;
        case ngx_signal_value(NGX_SHUTDOWN_SIGNAL):
            ngx_quit = 1;
            action = ", shutting down";
            break;

        case ngx_signal_value(NGX_TERMINATE_SIGNAL):
        case SIGINT:
            ngx_terminate = 1;
            action = ", exiting";
            break;

        case ngx_signal_value(NGX_REOPEN_SIGNAL):
            ngx_reopen = 1;
            action = ", reopening logs";
            break;

        case ngx_signal_value(NGX_RECONFIGURE_SIGNAL):
        case ngx_signal_value(NGX_CHANGEBIN_SIGNAL):
        case SIGIO:
            action = ", ignoring";
            break;
        }

        break;
    }

    ngx_log_error(NGX_LOG_NOTICE, ngx_cycle->log, 0,
                  "signal %d (%s) received%s", signo, sig->signame, action);

    if (ignore) {
        ngx_log_error(NGX_LOG_CRIT, ngx_cycle->log, 0,
                      "the changing binary signal is ignored: "
                      "you should shutdown or terminate "
                      "before either old or new binary's process");
    }

    if (signo == SIGCHLD) {
        ngx_process_get_status();
    }

    ngx_set_errno(err);
}
</pre>
	
创建worker子进程的函数是ngx_start_worker_processes。
<pre>
static void
ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_t n, ngx_int_t type)
{
    ngx_int_t      i;
    ngx_channel_t  ch;

    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "start worker processes");

    ch.command = NGX_CMD_OPEN_CHANNEL; //传递给其他worker子进程的命令：打开通信管道

    //创建n个worker进程
    for (i = 0; i < n; i++) {

        //创建worker子进程并初始化相关资源和属性
        ngx_spawn_process(cycle, ngx_worker_process_cycle,
                          (void *) (intptr_t) i, "worker process", type);

        /*master父进程向所有已经创建的worker子进程（不包括本子进程）广播消息
        包括当前worker子进程的进程id、在进程表中的位置和管道句柄，这些worker子进程收到消息后，会更新这些消息到自己进程空间的进程表，以此实现两个worke子进程之间的通信。
        */
        ch.pid = ngx_processes[ngx_process_slot].pid;
        ch.slot = ngx_process_slot;
        ch.fd = ngx_processes[ngx_process_slot].channel[0];

        ngx_pass_open_channel(cycle, &ch);
    }
}
</pre>
这里ngx_pass_open_channel，即遍历所有worker进程，跳过自己和异常的worker，把消息发送给各个worker进程。worker进程的管道可读事件捕捉函数是ngx_channel_handler(ngx_event_t *ev)，在这个函数中，会读取message，然后解析，并根据不同给的命令做不同的处理。

这里有一个关键的函数是ngx_pid_t ngx_spawn_process(ngx_cycle_t *cycle, ngx_spawn_proc_pt proc, void *data,char *name, ngx_int_t respawn)。proc是子进程的执行函数，data是其参数，name是进程名。

这个函数的任务：

1. 有一个ngx_processes全局数组，包含了所有的子进程，这里会fork出子进程并放入相应的位置，并设置这个进程的相关属性。
2. 创建socketpair，并设置相关属性
3. 子啊子进程中执行传递进来的函数。

<pre>
u_long     on;
ngx_pid_t  pid;
ngx_int_t  s; //fork的子进程在ngx_processes中的位置

//如果传递进来的respawn>0，说明是要替换进程ngx_processes[respawn]，可安全重用该进程表项。
if (respawn >= 0) {
   s = respawn;
} else {
   遍历进程表，找到空闲的slot
   for (s = 0; s < ngx_last_process; s++) {
        if (ngx_processes[s].pid == -1) {
                break;
            }
        }

		//达到最大进程限制报错
        if (s == NGX_MAX_PROCESSES) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                          "no more than %d processes can be spawned",
                          NGX_MAX_PROCESSES);
            return NGX_INVALID_PID;
        }
    }
</pre> 

接下来创建一对socketpair句柄，然后初始化相关属性。

<pre>
	if (respawn != NGX_PROCESS_DETACHED) { //NGX_PROCESS_DETACHED是热代码替换

        /* Solaris 9 still has no AF_LOCAL */

 		//建立socketpair
        if (socketpair(AF_UNIX, SOCK_STREAM, 0, ngx_processes[s].channel) == -1)
        {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "socketpair() failed while spawning \"%s\"", name);
            return NGX_INVALID_PID;
        }

        。。。

		//设置非阻塞模式
        if (ngx_nonblocking(ngx_processes[s].channel[0]) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          ngx_nonblocking_n " failed while spawning \"%s\"",
                          name);
            ngx_close_channel(ngx_processes[s].channel, cycle->log);
            return NGX_INVALID_PID;
        }

        if (ngx_nonblocking(ngx_processes[s].channel[1]) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          ngx_nonblocking_n " failed while spawning \"%s\"",
                          name);
            ngx_close_channel(ngx_processes[s].channel, cycle->log);
            return NGX_INVALID_PID;
        }

        //打开异步模式
        on = 1;
        if (ioctl(ngx_processes[s].channel[0], FIOASYNC, &on) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "ioctl(FIOASYNC) failed while spawning \"%s\"", name);
            ngx_close_channel(ngx_processes[s].channel, cycle->log);
            return NGX_INVALID_PID;
        }

		//设置异步io所有者
        if (fcntl(ngx_processes[s].channel[0], F_SETOWN, ngx_pid) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "fcntl(F_SETOWN) failed while spawning \"%s\"", name);
            ngx_close_channel(ngx_processes[s].channel, cycle->log);
            return NGX_INVALID_PID;
        }

        //当exec后关闭句柄
        if (fcntl(ngx_processes[s].channel[0], F_SETFD, FD_CLOEXEC) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "fcntl(FD_CLOEXEC) failed while spawning \"%s\"",
                           name);
            ngx_close_channel(ngx_processes[s].channel, cycle->log);
            return NGX_INVALID_PID;
        }

        if (fcntl(ngx_processes[s].channel[1], F_SETFD, FD_CLOEXEC) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "fcntl(FD_CLOEXEC) failed while spawning \"%s\"",
                           name);
            ngx_close_channel(ngx_processes[s].channel, cycle->log);
            return NGX_INVALID_PID;
        }

 		//设置当前子进程的句柄
        ngx_channel = ngx_processes[s].channel[1];

    } else {
        ngx_processes[s].channel[0] = -1;
        ngx_processes[s].channel[1] = -1;
    }
</pre>

接下来就是fork子进程，并设置进程相关参数。
<pre>
	//设置进程在进程表中的slot
	ngx_process_slot = s;

	pid = fork();

    switch (pid) {

    case -1:
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "fork() failed while spawning \"%s\"", name);
        ngx_close_channel(ngx_processes[s].channel, cycle->log);
        return NGX_INVALID_PID;

    case 0:
    	//子进程，执行传递进来的子进程函数
        ngx_pid = ngx_getpid();
        proc(cycle, data);
        break;

    default:
        break;
    }

    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "start %s %P", name, pid);

    ngx_processes[s].pid = pid;
    ngx_processes[s].exited = 0;

    if (respawn >= 0) { //使用原来的子进程即可
        return pid;
    }

    //初始化进程结构
    ngx_processes[s].proc = proc;
    ngx_processes[s].data = data;
    ngx_processes[s].name = name;
    ngx_processes[s].exiting = 0;

    switch (respawn) {

    case NGX_PROCESS_NORESPAWN:
        ngx_processes[s].respawn = 0;
        ngx_processes[s].just_spawn = 0;
        ngx_processes[s].detached = 0;
        break;

    case NGX_PROCESS_JUST_SPAWN:
        ngx_processes[s].respawn = 0;
        ngx_processes[s].just_spawn = 1;
        ngx_processes[s].detached = 0;
        break;

    case NGX_PROCESS_RESPAWN:
        ngx_processes[s].respawn = 1;
        ngx_processes[s].just_spawn = 0;
        ngx_processes[s].detached = 0;
        break;

    case NGX_PROCESS_JUST_RESPAWN:
        ngx_processes[s].respawn = 1;
        ngx_processes[s].just_spawn = 1;
        ngx_processes[s].detached = 0;
        break;

    case NGX_PROCESS_DETACHED:
        ngx_processes[s].respawn = 0;
        ngx_processes[s].just_spawn = 0;
        ngx_processes[s].detached = 1;
        break;
    }

    if (s == ngx_last_process) {
        ngx_last_process++;
    }	
</pre>

最后看一下，worker进程执行的函数static void
ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)。

1. 调用ngx_worker_process_init初始化；
	
	* 设置ngx_process=NGX_PROCESS_WORKER
	* 全局性的设置，包括执行环境、优先级、限制、setgid、setuid、信号初始化
	* 调用所有模块的init_process钩子
	* 关闭不使用的管道句柄，关闭当前的worker子进程的channel[0]句柄和继承来的其他进程的channel[1]句柄。使用其他进程的channel[0]句柄发送消息，使用本进程的channel[1]句柄监听事件。
	
2. 进行线程相关的操作。（如果有线程模式）
3. 主循环处理各种状态，类似master进程的主循环。

<pre>	
for ( ;; ) {

        if (ngx_exiting) { //退出状态，关闭所有连接

            c = cycle->connections;

            for (i = 0; i < cycle->connection_n; i++) {

                /* THREAD: lock */

                if (c[i].fd != -1 && c[i].idle) {
                    c[i].close = 1;
                    c[i].read->handler(c[i].read);
                }
            }

            if (ngx_event_timer_rbtree.root == ngx_event_timer_rbtree.sentinel)
            {
                ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");

                ngx_worker_process_exit(cycle);
            }
        }

        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "worker cycle");

        ngx_process_events_and_timers(cycle); //处理事件和计时

        if (ngx_terminate) { //收到NGX_CMD_TERMINATE命令，清理进城后退出，并调用所有模块的exit_process钩子。
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");

            ngx_worker_process_exit(cycle);
        }

        if (ngx_quit) {
            ngx_quit = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0,
                          "gracefully shutting down");
            ngx_setproctitle("worker process is shutting down");

            if (!ngx_exiting) {
            	//关闭监听socket，并设置退出状态
                ngx_close_listening_sockets(cycle);
                ngx_exiting = 1;
            }
        }

        if (ngx_reopen) { //重新打开log
            ngx_reopen = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reopening logs");
            ngx_reopen_files(cycle, -1);
        }
    }
</pre>

总结一下，nginx的启动过程可以划分为两个部分，

1. 读取配置文件并设置全局的配置结构信息以及每个模块的配置结构信息，调用模块的 create_conf钩子和init_conf钩子。
2. 创建进程和进程间通信机制，master进程负责管理各个worker子进程，通过 socketpair向子进程发送消息，各个worker子进程服务利用事件机制处理请求，通过socketpair与其他子进程通信（发送消息或者接收消息），进程启动的各个适当时机会调用模块的init_module钩子、init_process钩子、exit_process钩子和 exit_master钩子，init_master钩子没有被调用过。nginx的worker子进程继承了父进程的全局变量之后，子进程和父进程就会独立处理这些全局变量，有些全局量需要在父子进程之间同步就要通过通信方式了，比如 ngx_processes（进程表）的同步。