##其他IPC机制——信号

####信号集操作函数

sigset_t类型对于每种信号用一个bit表示“有效”或“无效”状态，至于这个类型内部如何存储这些bit则依赖于系统实现，从使用者的角度是不必关心的，使用者只能调用以下函数来操作sigset_t变量，而不应该对它的内部数据做任何解释，比如用printf直接打印sigset_t变量是没有意义的。

```C
#include <signal.h>

int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
int sigaddset(sigset_t *set, int signo);
int sigdelset(sigset_t *set, int signo);
int sigismember(const sigset_t *set, int signo);
```

**函数使用说明**：函数sigemptyset初始化set所指向的信号集，使其中所有信号的对应bit清零，表示该信号集不包含任何有效信号。函数sigfillset初始化set所指向的信号集，使其中所有信号的对应bit置位，表示该信号集的有效信号包括系统支持的所有信号。注意，在使用sigset_t类型的变量之前，一定要调用sigemptyset或sigfillset做初始化，使信号集处于确定的状态。初始化sigset_t变量之后就可以在调用sigaddset和sigdelset在该信号集中添加或删除某种有效信号。这四个函数都是成功返回0，出错返回-1。sigismember是一个布尔函数，用于判断一个信号集的有效信号中是否包含某种信号，若包含则返回1，不包含则返回0，出错返回-1。

#####sigprocmask

调用函数sigprocmask可以读取或更改进程的信号屏蔽字。

```C
#include <signal.h>
int sigprocmask(int how, const sigset_t *set, sigset_t *oset);
```

返回值：若成功则为0，若出错则为-1

如果oset是非空指针，则读取进程的当前信号屏蔽字通过oset参数传出。如果set是非空指针，则更改进程的信号屏蔽字，参数how指示如何更改。如果oset和set都是非空指针，则先将原来的信号屏蔽字备份到oset里，然后根据set和how参数更改信号屏蔽字。假设当前的信号屏蔽字为mask，下表说明了how参数的可选值。
how参数的含义如下表:

|取值|含义|
|---|:---:|
|SIG_BLOCK|set包含了我们希望添加到当前信号屏蔽字的信号，相当于mask=mask按位或set|
|SIG_UNBLOCK|set包含了我们希望从当前信号屏蔽字中解除阻塞的信号，相当于mask=mask&~set|
|SIG_SETMASK|设置当前信号屏蔽字为set所指向的值，相当于mask=set|

例程如下所示：

```C
#include<signal.h>
#include<stdio.h>
/* Handler function */
void
handler (int sig)
{
  printf ("Receive signal: %u\n", sig);
};

int
main (void)
{
  struct sigaction sa;
  int count;
/* Initialize the signal handler structure */
  sa.sa_handler = handler;
  /*以下注释基于sigprocmask (SIG_SETMASK, &sa.sa_mask, &oset);该条语句*/
  sigemptyset (&sa.sa_mask);//允许接收所有信号量
  sigaddset (&sa.sa_mask, SIGTERM);//屏蔽SIGTERM信号
  sigaddset (&sa.sa_mask, 1);////屏蔽signum为1的信号
  sigset_t oset;
  printf ("sa_mask value:%lu\n", sa.sa_mask.__val[0]);
  sa.sa_flags = 0;
/* Assign a new handler function to the SIGTERM signal */
  sigaction (SIGTERM, &sa, NULL);
  sigprocmask (SIG_SETMASK, &sa.sa_mask, &oset);	/* Accept all signals */
  printf ("sa_mask value:%lu\n", oset.__val[0]);
  printf ("sa_mask value:%lu\n", oset.__val[1]);
  sigprocmask (SIG_SETMASK, &sa.sa_mask, &oset);	/* Accept all signals */
  printf ("sa_mask value:%lu\n", oset.__val[0]);
  printf ("sa_mask value:%lu\n", oset.__val[1]);
/* Block and wait until a signal arrives */
  while (1)
    {
      sigsuspend (&sa.sa_mask);
      printf ("loop\n");
    }
  return 0;
}
```

==输出结果：==  

![结果1](/images/posts/signalResO.png)  
因为在main函数设计中阻塞了SIGTERM(15)和1的信号，所以当发送`kill -SIGTERM 16320`和`kill -1 16320`时，进程收不到信号（被阻塞了）。

![结果2](/images/posts/signalResT.png)  
因为在main函数设计中没有阻塞了4信号，所以当发送`kill -4 16320`时，进程可以收到信号。
####参考资料
* [阻塞信号](http://zyan.cc/book/linux_c/html/ch33s03.html)
