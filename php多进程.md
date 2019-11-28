PHP多进程及信号处理
0. 前言
可能在php中使用到多进程的地方比较少，但不妨碍我们去理解他的思路。各个语言的多进程方法基本是相通的，比如我们常用的nginx
通过php去探索一下多进程
从2道面试说起
nginx如何平滑重启？
nginx -s reload
平滑重启的原理是什么？
php-fpm如何平滑重启？
kill -USR2 {php-fpm-master-pid}
php-fpm真的是平滑重启吗？
1.多进程
1.1 多进程的实现
多进程的实现方式: fork(php中需要pcntl扩展库)
fork函数被调用一次，但返回两次。子进程中的返回值是0，父进程的返回值则是新子进程的进程ID（>0）
子进程会获得父进程数据空间、堆栈的副本
$glob = 1;
fork();
function fork()
{
    global $glob;
    $var = 1;
    $pid = pcntl_fork();
    if ($pid > 0) {
        echo sprintf("这里是主进程: %d, 子进程id: %d\n", getmypid(), $pid);
        $var++;
        $glob++;
    } else if ($pid == 0) {
        echo sprintf("这里是子进程: %d\n", getmypid());
        $var++;
        $glob++;
    } else {
        exit("fork fail");
    }

    echo sprintf("pid: %d, var=%d, glob=%d\n", getmypid(), $var, $glob);
}
输出的结果：

这里是主进程: 83348, 子进程id: 83349
pid: 83348, var=2, glob=2
这里是子进程: 83349
pid: 83349, var=2, glob=2

1.2 僵死(zombie)进程
定义：
在UNIX术语中，一个已经终止、但是其父进程尚未对其进行善后处理(获取终止子进程的有关信息，释放它仍占用的资源)的进程被称为僵死进程（zombie）。

如何写出僵死进程
function zChildProcess()
{
    $pid = pcntl_fork();
    if ($pid == -1) {
        exit("fork fail");
    } elseif ($pid) {
        $id = getmypid();
        echo "Parent process,pid {$id}, child pid {$pid}\n";

        while (1) {
            sleep(3);
        }
    } else {
        $id = getmypid();
        echo "Child process,pid {$id}\n";
        sleep(2);
        //子进程退出
        exit();
    }
}

//查找僵死进程
ps -A -o stat,ppid,pid,cmd | grep -e '^[Zz]'
如何避免僵死进程？
主进程主动wait子进程的退出
$childCount = 0;
$maxProcessNum = 100;
for ($i = 0; $i < $maxProcessNum; $i++) {
    $pid = pcntl_fork();

    if ($pid > 0) {
        //主进程
        $childCount++;
    } else if ($pid == 0) {
        //子进程
        //do something...
        exit();
    }
}
$status = null;
while ($childCount--) {
    //主进程等待子进程退出，避免出现僵死进程
    pcntl_wait($status);
}
交给1号进程（下一节的孤儿进程）
1.3 孤儿进程
一个其父进程已经终止的进程称为孤儿进程，这种进程由init进程（pid为1）收养
function OrphanProcess()
{
    $pid = pcntl_fork();
    if ($pid == -1) {
        exit("fork fail");
    } elseif ($pid) {       //父进程
        $id = getmypid();
        echo "Parent process,pid {$id}, child pid {$pid}\n";
        sleep(10);
        exit();
    } else {                //子进程
        $id = getmypid();
        echo "Child process,pid {$id}\n";
        $i = 100;
        while ($i--) {
            sleep(3);
        }
        exit();
    }
}

// 通过ps查看，父进程会变成1号进程
主进程比子进程先结束运行，子进程变成1号进程的子进程，当子进程退出时，1号进程会处理它的善后工作，避免变为僵死进程
1.4 守护进程(daemon)
为什么启动nginx、redis后，并没有添加 & 符号，他就能在后台执行？
末尾添加添加 & 后，程序能够在后台运行，属于一个job，通过fg命令还能使其恢复到前台执行
守护进程是一直在后台执行，nginx、redis等就是通过daemon方式实现的后台运行
如何写出daemon进程
调用umask将文件模式创建屏蔽字设置为0，避免继承得来的文件模式屏蔽字
调用fork，使父进程退出
调用setsid()创建一个新的会话，和原来的终端脱离
将当前工作目录改为根目录
关闭不需要的文件描述符
将标准输入、标准输出、标准出错指向/dev/null
以上不是都必须的，但fork、setsid()是必须的，甚至有些会fork两次， 先看一下各种软件中是如何实现daemon的，然后再解释为何fork、setsid是必须的

workman中的daemon实现
protected static function daemonize()
{
    //……
    umask(0);
    $pid = pcntl_fork();
    if (-1 === $pid) {
        throw new Exception('fork fail');
    } elseif ($pid > 0) {
        //主进程第一次退出
        exit(0);
    }
    // 将当前进程选举为进程组的leader
    if (-1 === posix_setsid()) {
        throw new Exception("setsid fail");
    }
    // Fork again avoid SVR4 system regain the control of terminal.
    $pid = pcntl_fork();
    if (-1 === $pid) {
        throw new Exception("fork fail");
    } elseif (0 !== $pid) {
        exit(0);
    }
    
    //继续做其它事情
}
看一下nginx的daemon处理
// src/os/unix/ngx_daemon.c

ngx_int_t
ngx_daemon(ngx_log_t *log)
{
    int  fd;

    switch (fork()) {
    case -1:
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "fork() failed");
        return NGX_ERROR;

    case 0:
        break;

    default:
        exit(0);
    }

    ngx_pid = ngx_getpid();

    if (setsid() == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "setsid() failed");
        return NGX_ERROR;
    }

    umask(0);

    fd = open("/dev/null", O_RDWR);
    if (fd == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                      "open(\"/dev/null\") failed");
        return NGX_ERROR;
    }

    if (dup2(fd, STDIN_FILENO) == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "dup2(STDIN) failed");
        return NGX_ERROR;
    }

    if (dup2(fd, STDOUT_FILENO) == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "dup2(STDOUT) failed");
        return NGX_ERROR;
    }

#if 0
    if (dup2(fd, STDERR_FILENO) == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "dup2(STDERR) failed");
        return NGX_ERROR;
    }
#endif

    if (fd > STDERR_FILENO) {
        if (close(fd) == -1) {
            ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "close() failed");
            return NGX_ERROR;
        }
    }

    return NGX_OK;
}

redis的daemonize
// ./src/server.c
void daemonize(void) {
    int fd;

    if (fork() != 0) exit(0); /* parent exits */
    setsid(); /* create a new session */

    /* Every output goes to /dev/null. If Redis is daemonized but
     * the 'logfile' is set to 'stdout' in the configuration file
     * it will not log at all. */
    if ((fd = open("/dev/null", O_RDWR, 0)) != -1) {
        dup2(fd, STDIN_FILENO);
        dup2(fd, STDOUT_FILENO);
        dup2(fd, STDERR_FILENO);
        if (fd > STDERR_FILENO) close(fd);
    }
}
setsid()函数的定义

首先有一个进程组的概念，比如执行管道命令 ps -ef | grep xxx | wc -l,这些进程是一个进程组，每个进程组有一个组长，组成的pid等于组id
如果某个进程是一个组长，调用setsid()会返回错误
如果这个进程不是组长，则会创建一个新的会话，这个进程会变成新的进程组的组长
新组长没有控制终端，如果调用setsid()之前有控制终端，也会脱离（跟原来的终端断开联系）
守护进程的主要目的就是和终端断开联系，所以要调用一次setsid(),而fork之后的子进程一定不是进程组的组长，因此调用setsid()不会报错
fork 2次，既断开终端，也不是进程组长，防止再次获得终端，更安全一点（不调用获取终端的函数，也不会获取终端）
1.5 守护进程和多进程关系
守护进程和多进程其实没有必然联系
大多数软件在使用多进程的时候，都是先化身daemon进程，在执行多进程的逻辑
2. 信号量
一道面试题：
如何杀死一个进程？一般都能答出kill。继续追问，你写一个程序，当被kill时，并不退出，该如何实现呢？为什么kill -9 一定能杀死进程？如何优雅的退出？

信号量 (不讨论太深)
一个进程收到一个信号与处理器收到一个中断请求可以说是一样的。信号是进程间通信机制中唯一的异步通信机制，一个进程不必通过任何操作来等待信号的到达，事实上，进程也不知道信号到底什么时候到达。进程之间可以互相通过系统调用kill发送软中断信号。内核也可以因为内部事件而给进程发送信号，通知进程发生了某个事件。信号机制除了基本通知功能外，还可以传递附加信息。

信号处理：
忽略此信号
捕捉信号：要通知内核在某种信号发生时调用一个用户函数
执行默认动作
//信号处理需要注册ticks才能生效，这里务必注意
//PHP5.4以上版本就不再依赖ticks了
declare(ticks = 1);

//设置信号处理函数
function sig_handler($signo)
{
    switch ($signo) {
        case SIGUSR1:
            echo "SIGUSR1\n";
            break;
        case SIGUSR2:
            echo "SIGUSR2\n";
            break;
        default:
            echo "unknow";
            break;
    }

}

//忽略信号
pcntl_signal(SIGINT, SIG_IGN);
//绑定信号处理函数
pcntl_signal(SIGUSR1, "sig_handler");
pcntl_signal(SIGUSR2, "sig_handler");

posix_kill(posix_getpid(), SIGUSR1);
posix_kill(posix_getpid(), SIGUSR2);
通过pcntl_signal_dispatch来分发信号（分发之后才处理）
echo "安装信号处理器...\n";
pcntl_signal(SIGHUP,  function($signo) {
     echo "信号处理器被调用\n";
});

echo "为自己生成SIGHUP信号...\n";
posix_kill(posix_getpid(), SIGHUP);

echo "分发...\n";
pcntl_signal_dispatch();

echo "完成\n";

//输出结果：

安装信号处理器...
为自己生成SIGHUP信号...
分发...
信号处理器被调用
完成
kill后边可以接数字，默认是kill -15

nginx中的信号

信号	用途
TERM, INT	Quick shutdown
QUIT	Graceful shutdown
USR1	Reopen the log files
如何优雅的退出？ 修改循环的条件为false
class SingalBase
{
    /**
     * 循环条件
     *
     * @var bool
     */
    protected $loop = true;

    /**
     * 
     */
    public function __construct()
    {
        //检测pcntl扩展
        if (!function_exists("pcntl_signal")) {
            throw new \Exception("no pcntl extension\n");
        }
        //信号安装
        $this->installSignal();
    }

    /**
     * 任务处理
     */
    public function handle()
    {
        echo "start...\n";
        while ($this->loop) {
            //信号分发
            //pcntl_signal_dispatch();
            //调用用户函数
            $this->doJob();
            //信号分发
            pcntl_signal_dispatch();
        }
    }

    /**
     * 处理逻辑
     */
    public function doJob()
    {
        echo "before sleep...\n";
        $i = 5;
        while($i-- > 0) {
            sleep(1);
        }
        //sleep(10);
        echo "after sleep...\n";

    }

    /**
     * 信号安装
     */
    protected function installSignal()
    {
        pcntl_signal(SIGTERM, array($this, 'sigalHndler'));
        pcntl_signal(SIGHUP, array($this, 'sigalHndler'));
        pcntl_signal(SIGINT, array($this, 'sigalHndler'));
        //不能捕获或者忽略, kill -9
        //pcntl_signal(SIGKILL, array($this, 'sigalHndler'));
        //忽略信号
        //pcntl_signal(SIGTERM, SIG_IGN);
    }


    /**
     * 信号处理函数
     * @param $signo
     */
    private function sigalHndler($signo)
    {
        switch ($signo) {
            case SIGTERM:
                echo "sigterm (kill / kill -15)\n";
                $this->quit();
                break;
            case SIGHUP:
                echo "sighup (close terminal)\n";
                $this->quit();
                break;
            case SIGINT:    // ctrl + c
                echo "sigterm (ctrl + c)\n";
                $this->quit();
                break;
            default:
                // 处理所有其他信号
        }

    }

    private function quit()
    {
        $this->loop = false;
        //do something other...
        echo "quit...\n";
    }
}
3. 多进程管理工具
supervisor在lumen中的使用即实现原理
supervisor的配置
http://www.supervisord.org/configuration.html
[stopsignal]

The signal used to kill the program when a stop is requested. This can be any of TERM, HUP, INT, QUIT, KILL, USR1, or USR2.

Default: TERM

Required: No.

Introduced: 3.0

=======================================
[autorestart]

Specifies if supervisord should automatically restart a process if it exits when it is in the RUNNING state. May be one of false, unexpected, or true. If false, the process will not be autorestarted.
lumen中的处理

信号处理
vendor/illuminate/queue/Worker.php
// ……
protected function listenForSignals()
{
    pcntl_async_signals(true);

    pcntl_signal(SIGTERM, function () {
        $this->shouldQuit = true;
    });

    pcntl_signal(SIGUSR2, function () {
        $this->paused = true;
    });

    pcntl_signal(SIGCONT, function () {
        $this->paused = false;
    });
}
……
主循环中会判断对应的状态：

/**
 * Listen to the given queue in a loop.
 *
 * @param  string  $connectionName
 * @param  string  $queue
 * @param  \Illuminate\Queue\WorkerOptions  $options
 * @return void
 */
public function daemon($connectionName, $queue, WorkerOptions $options)
{
    if ($this->supportsAsyncSignals()) {
        $this->listenForSignals();
    }

    $lastRestart = $this->getTimestampOfLastQueueRestart();

    while (true) {
        // ……

        // Finally, we will check to see if we have exceeded our memory limits or if
        // the queue should restart based on other indications. If so, we'll stop
        // this worker and let whatever is "monitoring" it restart the process.
        $this->stopIfNecessary($options, $lastRestart);
    }
}
autorestart
 /**
 * Stop the process if necessary.
 *
 * @param  \Illuminate\Queue\WorkerOptions  $options
 * @param  int  $lastRestart
 */
protected function stopIfNecessary(WorkerOptions $options, $lastRestart)
{
    if ($this->shouldQuit) {
        $this->kill();
    }

    if ($this->memoryExceeded($options->memory)) {
        $this->stop(12);
    } elseif ($this->queueShouldRestart($lastRestart)) {
        $this->stop();
    }
}


4. 如何自己实现多进程管理
流程图
一个完整的例子
<?php
/**
 *
 * User: chenxingyu<chenxingyu@cmcm.com>
 * Date: 2019-04-27 22:52
 */

$m = new Master(5);
$m->setChildMacRunTime(20);
$m->run();

class Master
{
    /**
     *
     * 最大子进程数量
     * @var int
     */
    private $maxChild;

    /**
     *
     * 子进程最大运行时间
     * @var int
     */
    private $childRunTime = 3600;

    /**
     *
     * 子进程
     * @var array
     */
    private $workers = [];

    /**
     *
     * 是否停止运行
     * @var bool
     */
    private $isStop = false;

    /**
     * Worker constructor.
     * @param int $maxChild
     */
    public function __construct($maxChild = 1)
    {
        $this->maxChild = $maxChild;
    }

    public function setChildMacRunTime($t)
    {
        $this->childRunTime = $t;
    }

    public function run()
    {
        //$this->daemonize();
        //创建子进程
        //$this->forkWorkers();
        //安装信号
        $this->installSignal();
        //管理子进程
        $this->monitorWorkers();
    }


    /**
     *  守护进程
     * @throws Exception
     */
    private function daemonize()
    {
        umask(0);
        $pid = pcntl_fork();
        if (-1 === $pid) {
            throw new Exception('fork fail');
        } elseif ($pid > 0) {
            exit(0);
        }
        if (-1 === posix_setsid()) {
            throw new Exception("setsid fail");
        }
        // Fork again avoid SVR4 system regain the control of terminal.
        $pid = pcntl_fork();
        if (-1 === $pid) {
            throw new Exception("fork fail");
        } elseif (0 !== $pid) {
            exit(0);
        }
    }

    /**
     * 管理子进程
     */
    private function monitorWorkers()
    {
        while (true) {
            pcntl_signal_dispatch();
            $status = 0;
            $pid = pcntl_wait($status, WUNTRACED);
            if ($pid > 0) {
                unset($this->workers[$pid]);
            }

            //如果没有stop，则继续运行
            if (!$this->isStop) {
                $this->forkWorkers();
            }
            //停止，并且子进程全部退出
            if ($this->isStop && count($this->workers) == 0) {
                break;
            }
        }
    }

    /**
     * 创建子进程
     */
    private function forkWorkers()
    {
        while (count($this->workers) < $this->maxChild) {
            $this->forkWorker();
        }
    }

    /**
     * fork一个子进程
     * @throws Exception
     */
    private function forkWorker()
    {
        $pid = pcntl_fork();
        if ($pid > 0) {      //住进
            $this->workers[$pid] = 1;
        } else if ($pid == 0) {      //子进程
            $w = new Worker($this->childRunTime);
            $w->run();
            exit;
        } else {
            throw new Exception("forkOneWorker fail");
        }
    }

    /**
     * 安装信号
     */
    public function installSignal()
    {
        pcntl_signal(SIGINT, array($this, 'signalHandler'));
        pcntl_signal(SIGTERM, array($this, 'signalHandler'));
        pcntl_signal(SIGUSR1, array($this, 'signalHandler'));
    }

    /**
     * 信号处理函数
     * @param $signal
     */
    public function signalHandler($signal)
    {
        switch ($signal) {
            // Stop.
            case SIGINT:
            case SIGTERM:
                $this->isStop = true;
                $this->stopWorkerGraceful();
                file_put_contents("1.log", "master get quit singal\n", FILE_APPEND | LOCK_EX);
                break;
            // Reload.
            case SIGUSR1:
                $this->stopWorkerGraceful();
                break;
        }
    }

    /**
     * 子进程平滑退出
     */
    private function stopWorkerGraceful()
    {
        foreach ($this->workers as $pid => $v) {
            posix_kill($pid, SIGTERM);
        }
    }
}


class Worker
{

    /**
     *
     * 开始运行的时间戳
     * @var int
     */
    private $startTime;

    /**
     *
     * 最长运行时间(seconds)， 0表示不退出
     * @var int
     */
    private $maxRunTime = 0;

    /**
     *
     * 是否停止运行
     * @var bool
     */
    private $isStop = false;

    /**
     * Worker constructor.
     */
    public function __construct($maxRunTime = 0)
    {
        $this->startTime = time();
        $this->maxRunTime = $maxRunTime;
        $this->installSignal();
    }

    /**
     * 处理
     */
    public function run()
    {
        while (!$this->isStop) {
            $this->doJob();
            $this->checkRuntime();
            pcntl_signal_dispatch();
        }
    }

    /**
     * 处理具体任务
     */
    private function doJob()
    {
        file_put_contents("1.log", sprintf("%s - %d - dojob...\n", date("Y-m-d H:i:s"), posix_getpid()), FILE_APPEND | LOCK_EX);
        $t = rand(1, 10);
        while ($t--) {
            sleep(1);
        }
    }

    private function checkRuntime()
    {
        if ($this->maxRunTime && ((time() - $this->startTime) > $this->maxRunTime)) {
            $this->isStop = true;
        }
    }

    /**
     * 安装信号
     */
    public function installSignal()
    {
        pcntl_signal(SIGINT, array($this, 'signalHandler'));
        pcntl_signal(SIGTERM, array($this, 'signalHandler'));
    }

    /**
     * 信号处理函数
     * @param $signal
     */
    public function signalHandler($signal)
    {
        switch ($signal) {
            // Stop.
            case SIGINT:
            case SIGTERM:
                //停止运行
                file_put_contents("1.log", sprintf("%s - %d - start quit...\n", date("Y-m-d H:i:s"), posix_getpid()), FILE_APPEND | LOCK_EX);
                $this->isStop = true;
                break;
        }
    }
}
5. 多进程的坑
子进程中要显式的exit()，避免执行主进程的逻辑
资源类文件要关闭，比如数据库、打开的文件句柄
避免把CPU占满
