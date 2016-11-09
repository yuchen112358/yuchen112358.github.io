## SESC源码阅读——System Call
***
#### 6.System Call
因为SESC不提供一个操作系统，所以SESC必须代表应用程序陷入系统调用并执行系统调用。对于每一个标准的系统调用，SESC将其转换为一个MINT函数。完整的系统调用及其对应的MINT函数如下：
```Java
//src/libmint/Subst.cpp
/* NOTE:
 *
 * It is NOT necessary to add underscores (_+) or libc_ at the
 * beginning of the function name. To avoid replication, noun_strcmp
 * eats all the underscores and libc_ that it finds.
 *
 * Ex: mmap would match for mmap, _mmap, __mmap, __libc_mmap ....
 *
 */

func_desc_t Func_subst[] = {
  // char *name               PFPI func;
  {"ulimit",                  mint_ulimit,                     1, OpExposed},
  {"execl",                   mint_execl,                      1, OpExposed},
  {"execle",                  mint_execle,                     1, OpExposed},
  {"execlp",                  mint_execlp,                     1, OpExposed},
  {"execv",                   mint_execv,                      1, OpExposed},
  {"execve",                  mint_execve,                     1, OpExposed},
  {"execvp",                  mint_execvp,                     1, OpExposed},
  {"mmap",                    mint_mmap,                       1, OpUndoable},
  {"open64",                  mint_open,                       1, OpUndoable}, 
  {"open",                    mint_open,                       1, OpUndoable},
  {"close",                   mint_close,                      1, OpUndoable},
  {"read",                    mint_read,                       1, OpUndoable},
  {"write",                   mint_write,                      1, OpUndoable},
  {"readv",                   mint_readv,                      1, OpExposed},
  {"writev",                  mint_writev,                     1, OpExposed},
  {"creat",                   mint_creat,                      1, OpExposed},
  {"link",                    mint_link,                       1, OpExposed},
  {"unlink",                  mint_unlink,                     1, OpExposed},
  {"rename",                  mint_rename,                     1, OpExposed},
  {"chdir",                   mint_chdir,                      1, OpExposed},
  {"chmod",                   mint_chmod,                      1, OpExposed},
  {"fchmod",                  mint_fchmod,                     1, OpExposed},
  {"chown",                   mint_chown,                      1, OpExposed},
  {"fchown",                  mint_fchown,                     1, OpExposed},
  {"lseek64",                 mint_lseek64,                    1, OpExposed},
  {"lseek",                   mint_lseek,                      1, OpExposed},
  {"access",                  mint_access,                     1, OpInternal},
  {"stat64",                  mint_stat64,                     1, OpExposed},
  {"lstat64",                 mint_lstat64,                    1, OpExposed},
  {"fstat64",                 mint_fstat64,                    1, OpExposed},
  {"xstat64",                 mint_xstat64,                    1, OpExposed},
  {"fxstat64",                mint_fxstat64,                   1, OpInternal},
  {"stat",                    mint_stat,                       1, OpExposed},
  {"lstat",                   mint_lstat,                      1, OpExposed},
  {"fstat",                   mint_fstat,                      1, OpExposed},
  {"xstat",                   mint_xstat,                      1, OpExposed},
  {"dup",                     mint_dup,                        1, OpExposed},
  {"pipe",                    mint_pipe,                       1, OpExposed},
  {"symlink",                 mint_symlink,                    1, OpExposed},
  {"readlink",                mint_readlink,                   1, OpExposed},
  {"umask",                   mint_umask,                      1, OpExposed},
  {"getuid",                  mint_getuid,                     0, OpInternal},
  {"geteuid",                 mint_geteuid,                    0, OpInternal},
  {"getgid",                  mint_getgid,                     0, OpInternal},
  {"getegid",                 mint_getegid,                    0, OpInternal},
  {"gethostname",             mint_gethostname,                1, OpExposed},
  {"getdomainname",           mint_getdomainname,              1, OpExposed},
  {"setdomainname",           mint_setdomainname,              1, OpExposed},
  {"socket",                  mint_socket,                     1, OpExposed},
  {"connect",                 mint_connect,                    1, OpExposed},
  {"send",                    mint_send,                       1, OpExposed},
  {"sendto",                  mint_sendto,                     1, OpExposed},
  {"sendmsg",                 mint_sendmsg,                    1, OpExposed},
  {"recv",                    mint_recv,                       1, OpExposed},
  {"recvfrom",                mint_recvfrom,                   1, OpExposed},
  {"recvmsg",                 mint_recvmsg,                    1, OpExposed},
  {"getsockopt",              mint_getsockopt,                 1, OpExposed},
  {"setsockopt",              mint_setsockopt,                 1, OpExposed},
  {"select",                  mint_select,                     1, OpExposed},
  {"cachectl",                mint_null_func,                  1, OpExposed},
  {"oserror",                 mint_oserror,                    0, OpExposed},
  {"setoserror",              mint_setoserror,                 0, OpExposed},
  {"perror",                  mint_perror,                     1, OpExposed},
  {"times",                   mint_times,                      1, OpUndoable},
  {"getdtablesize",           mint_getdtablesize,              1, OpExposed},
  {"syssgi",                  mint_syssgi,                     1, OpExposed},
  {"time",                    mint_time,                       1, OpExposed},
  {"munmap",                  mint_munmap,                     1, OpUndoable},
  {"malloc",                  mint_malloc,                     1, OpUndoable},
  {"calloc",                  mint_calloc,                     1, OpUndoable},
  {"free",                    mint_free,                       1, OpUndoable},
  {"cfree",                   mint_free,                       1, OpUndoable},
  {"realloc",                 mint_realloc,                    1, OpExposed},
  {"getpid",                  mint_getpid,                     1, OpInternal},
  {"getppid",                 mint_getppid,                    1, OpExposed},
  {"clock",                   mint_clock,                      1, OpExposed},
  {"gettimeofday",            mint_gettimeofday,               1, OpExposed},
  {"sysmp",                   mint_sysmp,                      1, OpExposed},
  {"getdents",                mint_getdents,                   1, OpExposed},
  {"_getdents",               mint_getdents,                   1, OpExposed},
  {"sproc",                   mint_notimplemented,             1, OpExposed},
  {"sprocsp",                 mint_notimplemented,             1, OpExposed},
  {"m_fork",                  mint_notimplemented,             1, OpExposed},
  {"m_next",                  mint_notimplemented,             1, OpExposed},
  {"m_set_procs",             mint_notimplemented,             1, OpExposed},
  {"m_get_numprocs",          mint_notimplemented,             1, OpExposed},
  {"m_get_myid",              mint_notimplemented,             1, OpExposed},
  {"m_kill_procs",            mint_notimplemented,             1, OpExposed},
  {"m_parc_procs",            mint_notimplemented,             1, OpExposed},
  {"m_rele_procs",            mint_notimplemented,             1, OpExposed},
  {"m_sync",                  mint_notimplemented,             1, OpExposed},
  {"m_lock",                  mint_notimplemented,             1, OpExposed},
  {"m_unlock",                mint_notimplemented,             1, OpExposed},
  {"fork",                    mint_notimplemented,             1, OpExposed},
  {"wait",                    mint_notimplemented,             1, OpExposed},
  {"wait3",                   mint_notimplemented,             1, OpExposed},
  {"waitpid",                 mint_notimplemented,             1, OpExposed},
  {"pthread_create",          mint_notimplemented,             1, OpExposed},
  {"pthread_lock",            mint_notimplemented,             1, OpExposed},
  /* wait must generate a yield because it might block */
  {"isatty",                  mint_isatty,                     1, OpInternal},
  {"ioctl",                   mint_ioctl,                      1, OpExposed},
  {"prctl",                   mint_prctl,                      1, OpExposed},
  {"fcntl64",                 mint_fcntl,                      1, OpInternal},
  {"fcntl",                   mint_fcntl,                      1, OpInternal},
  {"cerror64",                mint_cerror,                     1, OpExposed},
  {"cerror",                  mint_cerror,                     1, OpExposed},
#if (defined TASKSCALAR) & (! defined ATOMIC)
  {"abort",                   mint_rexit,                      1, OpExposed},
  {"exit",                    mint_rexit,                      1, OpExposed},
#else
  {"abort",                   mint_exit,                       1, OpExposed},
/* {"exit",                    mint_exit,                       1, OpClass(OpUndoable,OpAtStart)}, */
#endif
  {"sesc_fetch_op",           mint_sesc_fetch_op,              1, OpExposed},
  {"sesc_unlock_op",          mint_sesc_unlock_op,             1, OpExposed},
  {"sesc_spawn",              mint_sesc_spawn,                 1, OpNoReplay},
  {"sesc_spawn_",             mint_sesc_spawn,                 1, OpNoReplay},
  {"sesc_sysconf",            mint_sesc_sysconf,               1, OpExposed},
  {"sesc_sysconf_",           mint_sesc_sysconf,               1, OpExposed},
  {"sesc_self",               mint_getpid,                     1, OpInternal},
  {"sesc_self_",              mint_getpid,                     1, OpInternal},
  {"sesc_exit",               mint_exit,                       1, OpExposed},
  {"sesc_exit_",              mint_exit,                       1, OpExposed},
  {"sesc_finish",             mint_finish,                     1, OpExposed},
  {"sesc_yield",              mint_sesc_yield,                 1, OpExposed},
  {"sesc_yield_",             mint_sesc_yield,                 1, OpExposed},
  {"sesc_suspend",            mint_sesc_suspend,               1, OpExposed},
  {"sesc_suspend_",           mint_sesc_suspend,               1, OpExposed},
  {"sesc_resume",             mint_sesc_resume,                1, OpExposed},
  {"sesc_resume_",            mint_sesc_resume,                1, OpExposed},
  {"sesc_simulation_mark",    mint_sesc_simulation_mark,       1, OpExposed},
  {"sesc_simulation_mark_id", mint_sesc_simulation_mark_id,    1, OpExposed},
  {"sesc_fast_sim_begin",     mint_sesc_fast_sim_begin,        1, OpExposed},
  {"sesc_fast_sim_begin_",    mint_sesc_fast_sim_begin,        1, OpExposed},
  {"sesc_fast_sim_end",       mint_sesc_fast_sim_end,          1, OpExposed},
  {"sesc_fast_sim_end_",      mint_sesc_fast_sim_end,          1, OpExposed},
  {"sesc_preevent",           mint_sesc_preevent,              1, OpExposed},
  {"sesc_preevent_",          mint_sesc_preevent,              1, OpExposed},
  {"sesc_postevent",          mint_sesc_postevent,             1, OpExposed},
  {"sesc_postevent_",         mint_sesc_postevent,             1, OpExposed},
  {"sesc_memfence",           mint_sesc_memfence,              1, OpExposed},
  {"sesc_memfence_",          mint_sesc_memfence,              1, OpExposed},
  {"sesc_acquire",            mint_sesc_acquire,               1, OpExposed},
  {"sesc_acquire_",           mint_sesc_acquire,               1, OpExposed},
  {"sesc_release",            mint_sesc_release,               1, OpExposed},
  {"sesc_release_",           mint_sesc_release,               1, OpExposed},
  {"sesc_wait",               mint_sesc_wait,                  1, OpClass(OpUndoable,OpAtStart)},
  {"sesc_pseudoreset",        mint_sesc_pseudoreset,           1, OpExposed},
  //  {"printf",                      mint_printf,                     1, OpExposed},
  //  {"IO_printf",                   mint_printf,                     1, OpExposed},
  {"sesc_get_num_cpus",       mint_sesc_get_num_cpus,          0, OpInternal},
#if (defined TLS)
  //  {"sesc_begin_epochs",       mint_sesc_begin_epochs,          0, OpExposed},
  {"sesc_future_epoch",       mint_sesc_future_epoch,          0, OpExposed},
  {"sesc_future_epoch_jump",  mint_sesc_future_epoch_jump,     0, OpExposed},
  {"sesc_commit_epoch",       mint_sesc_commit_epoch,          0, OpExposed},
  {"sesc_change_epoch",       mint_sesc_change_epoch,          0, OpInternal},
  //  {"sesc_end_epochs",         mint_sesc_end_epochs,            0, OpExposed},
#endif
#ifdef TASKSCALAR
  {"sesc_fork_successor",     mint_sesc_fork_successor,        0, OpExposed},
  {"sesc_prof_fork_successor",mint_sesc_prof_fork_successor,   0, OpExposed},
  {"sesc_commit",             mint_sesc_commit,                0, OpExposed},
  {"sesc_prof_commit",        mint_sesc_prof_commit,           0, OpExposed},
  {"sesc_become_safe",        mint_sesc_become_safe,           0, OpExposed},
  {"sesc_is_safe",            mint_sesc_is_safe,               0, OpExposed},
  {"sesc_is_versioned",       mint_do_nothing,                 0, OpExposed},
  {"sesc_begin_versioning",   mint_do_nothing,                 0, OpExposed},
#endif
#ifdef SESC_LOCKPROFILE
  {"sesc_startlock",          mint_sesc_startlock,             0, OpExposed},
  {"sesc_endlock",            mint_sesc_endlock,               0, OpExposed},
  {"sesc_startlock2",         mint_sesc_startlock2,            0, OpExposed},
  {"sesc_endlock2",           mint_sesc_endlock2,              0, OpExposed},
#endif
#ifdef VALUEPRED
  {"sesc_get_last_value",     mint_sesc_get_last_value,        0, OpExposed}, 
  {"sesc_put_last_value",     mint_sesc_put_last_value,        0, OpExposed}, 
  {"sesc_get_stride_value",   mint_sesc_get_stride_value,      0, OpExposed}, 
  {"sesc_put_stride_value",   mint_sesc_put_stride_value,      0, OpExposed}, 
  {"sesc_get_incr_value",     mint_sesc_get_incr_value,        0, OpExposed}, 
  {"sesc_put_incr_value",     mint_sesc_put_incr_value,        0, OpExposed}, 
  {"sesc_verify_value",       mint_sesc_verify_value,          0, OpExposed}, 
#endif
  {"uname",                   mint_uname,                      1, OpInternal},
  {"getrlimit",               mint_getrlimit,                  1, OpExposed},
  {"setrlimit",               mint_do_nothing,                 1, OpExposed},
  {"getrusage",               mint_getrusage,                  1, OpExposed},
  {"syscall_fstat64",         mint_fstat64,                    1, OpExposed},
  {"syscall_fstat",           mint_fstat,                      1, OpExposed},
  {"syscall_stat64",          mint_stat64,                     1, OpExposed},
  {"syscall_stat",            mint_stat,                       1, OpExposed},
  {"syscall_lstat64",         mint_lstat64,                    1, OpExposed},
  {"syscall_lstat",           mint_lstat,                      1, OpExposed},
  {"syscall_getcwd",          mint_getcwd,                     1, OpExposed},
  {"assert_fail",             mint_assert_fail,                1, OpExposed},
  {"sigaction",               mint_do_nothing,                 1, OpInternal},
  {"ftruncate64",             mint_do_nothing,                 1, OpExposed},
  {"rmdir",                   mint_rmdir,                      1, OpExposed},
  {"fxstat",                  mint_fxstat64,                   1, OpExposed},
  { NULL,                     NULL,                            1, OpExposed}
};
```
MINT模拟器模拟大部分系统调用。然而，有一些系统调用其不能模拟，如fork()和sproc()，这些系统调用需要和操作系统打交道，因为新产生的进程将被怎样调度是不清楚的。因为SESC不提供一个操作系统，libapp即模拟了上述需要需操作系统打交道的系统调用。

##### 6.1 Thread API
Thread API定义在`src/libapp/Sescapi.h`文件中。
```Java
//src/libapp/Sescapi.h
  void sesc_init(void);
  int32_t  sesc_get_num_cpus(void);
  void sesc_sysconf(int32_t tid, int32_t flags);
  int32_t  sesc_spawn(void (*start_routine) (void *), void *arg, int32_t flags);
  int32_t  sesc_self(void);
  int32_t  sesc_suspend(int32_t tid);
  int32_t  sesc_resume(int32_t tid);
  int32_t  sesc_yield(int32_t tid);
  void sesc_exit(int32_t err);
  void sesc_finish(void);  /* Finish the whole simulation */
  void sesc_wait(void);
```
```Java
•	void sesc_init(): Initializes the library and spawns the internal scheduler thread and transforms the single execution unit of the current process into a thread. It must be the first function call of the thread API that is called from an application.

•	sesc_spawn(void (*start routine) (void *),void *arg,long flags): Creates a new thread of control that executes concurrently with the calling thread. The new thread calls the function start routine passing arg to it as the first argument. It returns an unique thread ID. For more information about the flags, please check sescapi.h.(重要)

•	void sesc_wait(). Blocks until one of the child threads have finished. If there are no child threads, it returns automatically. Unlike traditional wait() calls, it does not return the pid of the thread that finished.

•	void sesc_self(). Returns the current thread ID.

•	int sesc_suspend(int pid). Suspends a thread whose pid is equal to the argument. The thread is transitioned to suspended state and it is removed from the instruction execution loop. The function returns true on success and false on errors. If the calling thread is the only one in running state, the simulation concludes.(重要)

•	int sesc_resume(int pid). Resumes a thread in suspended state by moving it to a running queue. A resumed thread is usually assigned to the same CPU that it was running on before it was suspended in order to minimize the number of cache misses. However, the flags specified in thread creation can change this policy. The function returns true on success and false on errors.(重要)

•	int sesc_yied(int pid). Explicitly yields execution control to the thread whose pid is passed as a parameter. If pid = -1, any thread can be dispatched. The function returns true when it succeeds or false when the pid specifies an invalid or not yet ready thread.(重要)

•	void sesc_exit(). Terminates the current thread.

```

* 举例如下：

```Swift
//src/

/*
 * Flags specification:
 *
 * flags are used in sesc_spawn and sesc_sysconf. Both cases share the
 * same flags structure, but some flag parameters are valid in some
 * cases.
 *
 * Since the pid has only 16 bits and flags are 32 bits, the lower 16
 * bits are a parameter to the flags.
 */

/* The created thread is not a allowed to migrate between CPUs on
 * context switchs.
 */
#define SESC_FLAG_NOMIGRATE  0x10000

/* The tid parameter (the lower 16 bits) indicate in which CPU this
 * thread is going to be mapped. If the flag SESC_SPAWN_NOMIGRATE is
 * also provided, the thread would always be executed in the same
 * CPU. (unless sesc_sysconf is modifies the flags status)
 */
#define SESC_FLAG_MAP        0x20000

#define SESC_FLAG_FMASK      0x8fff0000
#define SESC_FLAG_DMASK      0xffff

/* Example of utilization:
 *
 * tid = sesc_spawn(func,0,SESC_FLAG_NOMIGRATE|SESC_FLAG_MAP|7);
 * This would map the thread in the processor 7 for ever.
 *
 * sesc_sysconf(tid,SESC_FLAG_NOMIGRATE|SESC_FLAG_MAP|2);
 * Moves the previous thread (tid) to the processor 2
 *
 * sesc_sysconf(tid,SESC_FLAG_MAP|2); 
 * Keeps the thread tid in the same processor, but it is allowed to
 * migrate in the next context switch.
 *
 * tid = sesc_spawn(func,0,0);
 * Creates a thread and maps it to the processor an iddle processor if
 * possible.
 *
 * tid = sesc_spawn(func,0,SESC_FLAG_NOMIGRATE);
 * The same that the previous, but once assigned to a processor, it
 * never migrates.
 *
 */
 ```

 ```Java
//关于进程迁移
//void RunningProcs::switchOut(CPU_t id, ProcessId *proc)/迁出
//void RunningProcs::switchIn(CPU_t id, ProcessId *proc)/迁入
 void xuStats::doMigrate(int *curr, int *dst){     //migrate threads from curr to dst.

	int        cpuId;
	ProcessId  *proc;
	nMigrate++;
	
	for(int pid = 1; pid < xu_nThread; pid++){  //switch out
		if(curr[pid] != dst[pid] && !is_Done[pid]){ //thread i need to be migrated form core_curr[i] to core_dst[i]
			cpuId      = curr[pid];
			proc       = ProcessId::getProcessId(pid);
			osSim->cpus.switchOut(cpuId, proc); //switch out proc  from cpuId
		        outputDataThread(cpuId);             //output cpuid's data 
			is_Free[cpuId] = true;
		}
	}
	for(int pid = 1; pid < xu_nThread ; pid++){  //switch in
		if(curr[pid] != dst[pid]){      //thread i need to be migrated form core_curr[i] to core_dst[i]
			curr[pid] = dst[pid];   // update thread->cpu
			cpuId = curr[pid];
			if( !is_Done[pid]){
				proc       = ProcessId::getProcessId(pid);
				osSim->cpus.switchIn(cpuId, proc);
				is_Free[cpuId] = false;
				saveDataContext(cpuId);
			}
			arc_sigma[cpuId] = pid;  // update cpu -> thread
		}
	}
}
```

* 与迁入/迁出相关的SESC源码

```Java
//src/libcore/RunningProcs.cpp
void RunningProcs::switchIn(CPU_t id, ProcessId *proc)
{
  GProcessor *core=getProcessor(id);

  workingListAdd(core);

  proc->switchIn(id);
#ifdef TS_STALL  
  core->setStallUntil(globalClock+5);
#endif  
  core->switchIn(proc->getPid()); // Must be the last thing because it can generate a switch
}

void RunningProcs::switchOut(CPU_t id, ProcessId *proc) 
{
  GProcessor *core=getProcessor(id);
  Pid_t pid = proc->getPid();

  proc->switchOut(core->getAndClearnGradInsts(pid),
                  core->getAndClearnWPathInsts(pid));

  core->switchOut(pid);

  workingListRemove(core);
}
```

* 与调度相关的SESC源码

```Java
//src/libcore/RunningProcs.cpp
void RunningProcs::makeRunnable(ProcessId *proc)
{
  // The process must be in the InvalidState
  I(proc->getState()==InvalidState);
  // Now the process is runnable (but still not running)
  ProcessId *victimProc=proc->becomeReady();
  // Get the CPU where the process would like to run
  CPU_t cpu=proc->getCPU();
  // If there is a preferred CPU, try to schedule there
  if(cpu>=0){
    // Get the GProcessor of the CPU
    GProcessor *core=getProcessor(cpu);
    // Are there available flows on this CPU
    if(core->availableFlows()){
      // There is an available flow, grab it
      switchIn(cpu,proc);
      return;
    }
  }
  // Could not run the process on the cpu it wanted
  // If the process is not pinned to that processor, try to find another cpu
  if(!proc->isPinned()){
    // Find an available processor
    GProcessor *core=getAvailableProcessor();
    // If available processor found, run there
    if(core){
      switchIn(core->getId(),proc);
      return;
    }
  }
  // Could not run on an available processor
  // If there is a process to evict, switch it out and switch the new one in its place
  if(victimProc){
    I(victimProc->getState()==RunningState);
    // get the processor where victim process is running
    cpu=victimProc->getCPU();
    switchOut(cpu, victimProc);
    switchIn(cpu,proc);
    victimProc->becomeNonReady();
    makeRunnable(victimProc);
  }
  // No free processor, no process to evict
  // The new process has to wait its turn 
}

void RunningProcs::makeNonRunnable(ProcessId *proc)
{
  // It should still be running or ready
  I((proc->getState()==RunningState)||(proc->getState()==ReadyState));
  // If it is still running, remove it from the processor
  if(proc->getState()==RunningState){
    // Find the CPU where it is running
    CPU_t cpu=proc->getCPU();
    // Remove it from there
    switchOut(cpu, proc);
    // Set the state to InvalidState to make the process non-runnable
    proc->becomeNonReady();
    // Find another process to run on this cpu
    ProcessId *newProc=ProcessId::queueGet(cpu);
    // If a process has been found, switch it in
    if(newProc){
      switchIn(cpu,newProc);
    }
  }else{
    // Just set the state to InvalidState to make it non-runnable
    proc->becomeNonReady();
  }
}
```

##### 6.2 Synchronization API
```Java
//src/libapp/Sescapi.h
#ifndef SESCAPI_NATIVE
  /*
   * LOCK/UNLOCK operation
   * a simple spin lock
   */
  void sesc_lock_init(slock_t * lock);
  void sesc_lock(slock_t * lock);
  void sesc_unlock(slock_t * lock);
#endif

  /*
   * Barrier 
   * a two-phase centralized barrier
   */
  void sesc_barrier_init(sbarrier_t *barr);
  void sesc_barrier(sbarrier_t *barr, int32_t num_proc);

  /*
   * Semaphore
   * busy-wait semaphore
   */
  void sesc_sema_init(ssema_t *sema, int32_t initValue);
  void sesc_psema(ssema_t *sema);
  void sesc_vsema(ssema_t *sema);
```
```Java
•	void sesc_lock_init(slock t *lock): Initializes the lock lock.

•	void sesc_lock(slock t *lock): Acquires the lock lock.

•	void sesc_unlock(slock t *lock): Releases the lock lock.

Other synchronization primitives supported by SESC are barriers and semaphores:
•	void sesc barrier_init(sbarrier t *barr): Initializes a barrier barr.

•	void sesc_barrier(sbarrier t *barr, long num proc): Executes a barrier barr.

•	void sesc_sema_init(ssema t *sema, int initValue): Initializes the semaphore sema.

•	void sesc_psema(ssema t *sema): Signals the semaphore sema.

•	void sesc_vsema(ssema t *sema): Waits for the signal for the semaphore sema.
```




