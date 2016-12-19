## SESC源码阅读——Instruction types & Emulation
***
#### 1.Instruction types
在SESC中存在两种指令类型。第一种表示应用二进制中的实际指令，第二种表示流水线中流动的短期指令。
###### 1.1 Static Instructions
Static Instructions的实现类是Instruction.
Instruction是在SESC的初始化期间进行创建的。一个函数读取应用的二进制并将每一条指令译码为内部表示。这些内部表示可以使得指令执行更快。例子：`add r1,r2,3`。
```Java
//src/libll/Instruction.h
  static void MIPSDecodeInstruction(size_t index
                                    ,icode_ptr &picode
                                    ,InstType &opcode
                                    ,InstSubType &subCode
                                    ,MemDataSize &dataSize
                                    ,RegType &src1
                                    ,RegType &dest
                                    ,RegType &src2
                                    ,EventType &uEvent
                                    ,bool &BJCondLikely
                                    ,bool &guessTaken
                                    ,bool &jumpLabel);
```
静态指令对象包含源寄存器、指令类型（branch，load，store，integer arithmetic或者 floating point arithmetic）、执行确切指令的函数的指针（如浮点加）、下一条执行指令的指针、分支目标的指针（如果存在）等。

###### 1.2 Dynamic Instructions
动态指令是一个静态指令的明确的实例。每当一个静态指令被运行，一个动态指令便会被创建。例子：对应于静态指令`add r1,r2,3`的一个动态指令可能是r1中保存的值是10，r2中保存的值是-8，并且处理器所处的时钟周期点为32948。
动态指令的实现类是DInst。DInst有许多字段。

```Java
//src/libcore/Dinst.h
class DInst {
 public:
  // In a typical RISC processor MAX_PENDING_SOURCES should be 2
  static const int32_t MAX_PENDING_SOURCES=2;

private:

  static pool<DInst> dInstPool;

  DInstNext pend[MAX_PENDING_SOURCES];
  DInstNext *last;
  DInstNext *first;
  
  int32_t cId;

  // BEGIN Boolean flags
  bool loadForwarded;
  bool issued;
  bool executed;
  bool depsAtRetire;
  bool deadStore;
  bool deadInst;
  bool waitOnMemory;

  bool resolved; // For load/stores when the address is computer, for the rest of instructions when it is executed
  //......
  const Instruction *inst;
  VAddr vaddr;
  Resource    *resource;
  DInst      **RATEntry;
  FetchEngine *fetch;
  //......
  public:
  static int32_t currentID;
  int32_t ID; // static ID, increased every create (currentID). pointer to the
  // DInst may not be a valid ID because the instruction gets recycled
  //......
  CallbackBase *pendEvent;
  //......
}
```
* 动态指令通过连接前后依赖的指令来跟踪指令之间的依赖关系。
* 如上所示，DInst类中的一些变量用来跟踪一个动态指令是否被发射、执行或退出;属于哪个CPU；DInst的序号；运行时所需要的资源（如浮点乘法器）；以及一个DInst是否执行在一个误预测分支的错误路径上。

#### 2.Emulation
仿真器（emulation）受ExecutionFlow类的控制。ExecutionFlow的外层接口是executionPC()函数。executionPC()函数执行下一条指令并返回对应的DInst对象。

```Go
//src/libll/ExecutionFlow.h
class ExecutionFlow : public GFlow {
private:
//.....
public:
  void switchIn(int32_t i);
  void switchOut(int32_t i);

  int32_t currentPid(void) {
#if (defined MIPS_EMUL)
    if(!context)
      return -1;
    return context->getPid();
#else // (defined MIPS_EMUL)
    return thread.getPid();
#endif // Else (defined MIPS_EMUL)
  }
 DInst *executePC();
 void goRabbitMode(long long n2skip=0);
//......
}
```
**executePC()函数实际上调用了exeInst()函数。**
```Java
//src/libll/ExecutionFlow.cpp
DInst *ExecutionFlow::executePC()
{
//......
  DInst *dinst=0;
  // We will need the original picodePC later
  icode_ptr origPIcode=picodePC;
  I(origPIcode);
  // Need to get the pid here because exeInst may switch to another thread
  Pid_t     origPid=thread.getPid();
  // Determine whether the instruction has a delay slot
  bool hasDelay= ((picodePC->opflags) == E_UFUNC);
  bool isEvent=false;
//......
    if (picodePC->target->opflags == E_NO_SPEC && !rsesc_is_safe(origPid)) {
      rsesc_become_safe(origPid);
      return 0; // Nothing
    }else if (picodePC->target->func == mint_sesc_fork_successor) {
      exeInst();
      propagateDepsIfNeeded();
      
      exeInst();
      propagateDepsIfNeeded();
      hasDelay=false;
	  
//......

  int32_t vaddr = exeInst();
//......
  vaddr = exeInst();
//......
  // Execute the actual event (but do not time it)
  I(thread.getPid()==origPid);
  vaddr = exeInst();
  I(vaddr);
#ifdef TASKSCALAR
  propagateDepsIfNeeded();
  const Instruction *eInst = Instruction::getInst(origPIcode->target->instID);
  if (eInst->getEvent() == SpawnEvent) {
    eInst = Instruction::getInst(picodePC->instID);
    while (!eInst->isBJCond() ) {
      exeInst();
      propagateDepsIfNeeded();
      eInst = Instruction::getInst(picodePC->instID);
    }
    exeInst(); // Also the BJ itself
    propagateDepsIfNeeded();
  }
//......
}
```

* exeInst()函数的代码如下：


```Java
int32_t ExecutionFlow::exeInst(void)
{
  // Instruction address
  int32_t iAddr = picodePC->addr;
  // Instruction flags
  short opflags=picodePC->opflags;
  // For load/store instructions, will contain the virtual address of data
  // For other instuctions, set to non-zero to return success at the end
  VAddr dAddrV=1;
  // For load/store instructions, the Real address of data
  // The Real address is the actual address in the simulator
  //   where the data is found. With versioning, the Real address
  //   actually points to the correct version of the data item
  RAddr dAddrR=0;

#if (defined TLS)
  Pid_t currPid=thread.getPid();
  tls::Epoch::disableBlockRemoval();
#endif

#ifdef TASKSCALAR
  // is a context switch it should look for a new TaskContext
  TaskContext *tc = TaskContext::getTaskContext(thread.getPid());
  if (tc==0) {
    if(picodePC->func == mint_exit) {
      do{
        picodePC=(picodePC->func)(picodePC, &thread);
      }while(picodePC->addr==iAddr);
    }
    return 0;
  }
  I(tc);
  I(thread.getPid() != -1);

  RAddr origdAddrR=0;
#endif
  
  // For load/store instructions, need to translate the data address
  if(opflags&E_MEM_REF) {
    // Get the Virtual address
         dAddrV = (*((int32_t *)&thread.reg[picodePC->args[RS]])) + picodePC->immed;
    // Get the Real address
    dAddrR = thread.virt2real(dAddrV, opflags);

    
#ifdef TASKSCALAR
    if (dAddrR == 0)   // happens if spec thread had a bad addr translation
      return 0;
    I(tc); 
    I(tc->getVersionRef());
    if (!tc->getVersionRef()->isSafe() && gmos->TLBTranslate(dAddrV) == -1) {
      // Past:  tc->syncBecomeSafe();
      return 0; // Do not advance unsafe threads that thave a TLB miss
    }
#endif
    
#if (defined TLS)
    I(thread.getEpoch());
    if(opflags&E_READ)
      dAddrR=thread.getEpoch()->read(iAddr, opflags, dAddrV, dAddrR);
    if(opflags&E_WRITE)
      dAddrR=thread.getEpoch()->write(iAddr, opflags, dAddrV, dAddrR);
    if(!dAddrR){
      tls::Epoch::enableBlockRemoval();
      return 0;
    }
#endif
    
#ifdef TASKSCALAR
    if(opflags&E_READ){
      I(!(opflags&E_WRITE));
      
      dAddrR=tc->read(iAddr, opflags, dAddrR);
    }else if(opflags&E_WRITE) {
      I(!(opflags&E_READ));
      
      origdAddrR = dAddrR;
      dAddrR = tc->preWrite(dAddrR);
    }
#endif
    
    // Put the Real data address in the thread structure
    thread.setRAddr(dAddrR);
  }
  
  do{
#if (defined TLS)
    if(thread.getPid()!=currPid)
      break;
    I(thread.getEpoch());
    I(picodePC->getClass()!=OpExposed);
#endif
    picodePC=(picodePC->func)(picodePC, &thread);
#if (defined TLS)
    thread.setPCIcode(picodePC);
#endif
    I(picodePC);
  }while(picodePC->addr==iAddr);

  //  MSG("0x%x",iAddr);

#ifdef TASKSCALAR
  if(opflags&E_WRITE) {
    I(origdAddrR);
    I(restartVer==0);
    restartVer = tc->postWrite(iAddr, opflags, origdAddrR);
  }
  #endif

#if (defined TLS)
  I(!tls::Epoch::isBlockRemovalEnabled());
  tls::Epoch::enableBlockRemoval();
#endif

  return dAddrV;
}
```
由上述代码可知，exeInst()函数执行了一些检查，如指令地址是否是一个合法的地址，然后执行指令。每一个Instruction对象包含一个指向仿真该条指令函数的指针。该条指令的源和目的操作数也被包含在Instruction对象中。因此实际的执行是非常快速的。  
在ExecutionFlow类中，还存在一个可以在rabbit模式下执行的函数（The **rabbit model** does not model their timing）。在rabbit模式下，Instructions仅仅被仿真，时间模拟将不会操作，模拟器执行指令的速度是全模拟模式下的1000倍。

```Java
void ExecutionFlow::goRabbitMode(long long n2skip)
{
  int32_t nFastSims = 0;
  if( ev == FastSimBeginEvent ) {
    // Can't train cache in those cases. Cache only be train if the
    // processor did not even started to execute instructions
    trainCache = 0;
    nFastSims++;
  }else{
    I(globalClock==0);
  }

  if (n2skip) {
    I(!goingRabbit);
    goingRabbit = true;
  }

  nExec=0;
  
  do {
    ev=NoEvent;
    if( n2skip > 0 )
      n2skip--;

    nExec++;
//......
```
`./sesc.mem -ccombina.conf -w100000 combina_1212`该条命令即是快速跳过前100000指令的时间模拟。




















