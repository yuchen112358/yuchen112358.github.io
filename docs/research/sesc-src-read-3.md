## SESC源码阅读——Caches
***
#### 4.Caches
典型的现代微处理器中含有一个高层的指令高速缓冲存储器(I-cache)和一个高层的数据高速缓冲存储器(D-cache)。这两个cache也被称为L1 cache。在L1 cache之下，是相对较慢的L2 cache。在许多配置中，也存在一个off-die L3 cache(其比L2 cache更大、更慢)。caches拥有许多参数，SESC可以模拟不同种类的cache：
* cache大小
* 命中/缺失延迟
* 替换策略
* 缓存行(cache-line)大小
* 关联方式

可想而知，要模型化涉及cache的所有的延迟和事务，SESC中cache的实现必然是非常复杂的，而且使用了许多事件驱动回调（因为在cache和bus中的实际延迟是不可预测的，所以CallBack类及其子类使得程序设计者可以在将来某个特定的时间上调度函数的调用。也只有回调机制可以保证周期精确的模拟结果。系统维护一个回调队列，每个回调对象产生后便会被插入到该队列中。回调对象记录了调用对象、其调用的成员函数、函数参数、调用周期点等。）。
```Bash
例如：  

1) 一个L1的读缺失可能导致L1中一个脏的cache-line被写回到L2，并且仅仅当这些完成后L1 缺失才会被发送到L2。  

2) 然后，如果读缺失发生在L2中，L2需要经仲裁访问总线，最后发送缺失到内存，等等。  

3) 这其中的每一步都会导致一个固定的延迟和仲裁损失（对于同一资源，可能有许多请求，这时便需要仲裁，而仲裁会导致损失）。
```
##### 4.1 GProcessor类
```Java
//src/libcore/GProcessor.h
class GProcessor {
private:
protected:
  // Per instance data
  const CPU_t Id;
  const int32_t FetchWidth;
  const int32_t IssueWidth;
  const int32_t RetireWidth;
  const int32_t RealisticWidth;
  const int32_t InstQueueSize;
  bool InOrderCore;
  const size_t MaxFlows;
  const size_t MaxROBSize;

  GMemorySystem *memorySystem;

  FastQueue<DInst *> ROB;

  FastQueue<DInst *> replayQ;
  LDSTQ lsq;
  //......
}
```
每一个GProcessor都拥有一个MemorySystem对象。MemorySystem对象创建一个分层的cache结构，并作为一个适配器服务于GProcessor和最高层的Caches之间,如下图所示。  

![adapter](/images/posts/sesc-cache00.png)  

当GProcessor需要与MemorySystem交互时，它产生一个MemoryRequest对象（对于数据，其为DMemoryRequest对象；对于指令，其为IMemoryRequest对象）。

##### 4.2 DMemRequest类和IMemRequest类
```Go
//src/libcore/MemRequest.h
class DMemRequest : public MemRequest {
  // MemRequest specialized for dcache
 private:
  static pool<DMemRequest, true> actPool;
  friend class pool<DMemRequest, true>;

  void destroy();
  static void dinstAck(DInst *dinst, MemOperation memOp, TimeDelta_t lat);

 protected:
 public:
  static void create(DInst *dinst, GMemorySystem *gmem, MemOperation mop);

  VAddr getVaddr() const;
  void ack(TimeDelta_t lat);
};

class IMemRequest : public MemRequest {
  // MemRequest specialed for icache
 private:
  static pool<IMemRequest, true> actPool;
  friend class pool<IMemRequest, true>;

  IBucket *buffer;

  void destroy();

 protected:
 public:
  static void create(DInst *dinst, GMemorySystem *gmem, IBucket *buffer);

  VAddr getVaddr() const;
  void ack(TimeDelta_t lat);
};
```
DMemoryRequest比IMemoryRequest多了一个dinstAck()函数，其用来区分一个访问是读还是写。  
MemoryRequest对象通过单一例化函数DMemRequest::create()或IMemRequest::create()创建:
```Java
//src/libcore/MemRequest.cpp
void DMemRequest::create(DInst *dinst, GMemorySystem *gmem, MemOperation mop)
{
  // turn off address translation
  int32_t old_addr = dinst->getVaddr();

#if !((defined TRACE_DRIVEN)||(defined QEMU_DRIVEN))
#if (defined MIPS_EMUL)
  ThreadContext *context=dinst->context;
#else
  ThreadContext *context=ThreadContext::getMainThreadContext();
#endif
  if(!context->isValidDataVAddr(old_addr)){
    dinstAck(dinst, mop, 0);
    return;
  }
#endif

  DMemRequest *r = actPool.out();//从Pool中分配一个新的MemRequest对象

  I(dinst != 0);//dinst == 0，输出一些信息

  IS(r->acknowledged = false);//执行r->acknowledged = false赋值操作
  I(r->memStack.empty());//r->memStack不为空，输出信息
  r->currentClockStamp = (Time_t) -1;

  r->setFields(dinst, mop, gmem->getDataSource());//gmem是传进来的被访问的对象，该函数设置当前访问的内存对象currentMemObj
  r->dataReq = true;
  r->prefetch= false;
  r->priority = 0;
#ifdef TLS
  r->clearStall();
#endif
 
  int32_t ph_addr = gmem->getMemoryOS()->TLBTranslate(old_addr);
  if (ph_addr == -1) {
    gmem->getMemoryOS()->solveRequest(r);
    return;
  }


  r->setPAddr(ph_addr);
  r->access();//调用最高层cache对象的access()函数
}
```

```Java
static pool<DMemRequest, true> actPool;

template<class Ttype, bool noTimeCheck=false>
class pool {
//......
}
```

```Java
void MemRequest::access()
{
  currentMemObj->access(this);
}

//src/libcore/MemoryRequest.h
void setFields(DInst *d, MemOperation mop, MemObj *mo) {
    dinst = d;
    if(d) {
      if(d->getResource())
	gproc = d->getResource()->getCluster()->getGProcessor();
    } else {
      gproc = 0;
    }
    memOp = mop;
    currentMemObj = mo;//设置当前访问的内存对象
}
```

```Java
#define IN(aC)   do{ int32_t aRes=0; aC; if(!aRes) doassert(); }while(0)
#define I(aC)    do{                 if(!(aC)) doassert(); }while(0)
#define GIN(g,e) do{ if(g) IN(e); }while(0)
#define GI(g,e)  do{ if(g)  I(e); }while(0)
#define GIS(g,e) do{ if(g)   e;   }while(0)
#define ID(e)        e
#define ID2(e)        e
#define IS(e)    do{ e;           }while(0)
```
```Java
#ifdef NANASSERTFILE
#define doassert()               \
    do {                                   \
      fprintf(ASSERTSTREAM,"%s (%s),%s line %d failed for ",__FILE__,NANASSERTFILE,NanassertID,__LINE__); \
      fprintf(ASSERTSTREAM,"\n");          \
      ASSERTACTION;                        \
    }while(0)
#else
#define doassert()               \
    do {                                   \
      fprintf(ASSERTSTREAM,"%s,%s line %d failed for ",__FILE__,NanassertID,__LINE__);\
      fprintf(ASSERTSTREAM,"\n");          \
      ASSERTACTION;                        \
    }while(0)
#endif
```
DMemRequest::create()函数的参数是请求关联的DInst对象、将要访问的MemorySystem对象和操作类型(读或写)。  
create()函数分配一个新的MemRequest对象，并初始化它，最后调用最高层cache对象的access()函数。  

##### 4.3 Cache类
```Java
//src/libmem/Cache.cpp
void Cache::access(MemRequest *mreq) 
{
  mreq->setClockStamp((Time_t) - 1);
  if(mreq->getPAddr() <= 1024) { // TODO: need to implement support for fences
    mreq->goUp(0); 
    return;
  }

  nAccesses[getBankId(mreq->getPAddr())]->inc();

  switch(mreq->getMemOperation()){
  case MemReadW:
  case MemRead:    read(mreq);       break;
  case MemWrite:   write(mreq);      break;
  case MemPush:    pushLine(mreq);   break;
  default:         specialOp(mreq);  break;
  }
}

//读

void Cache::read(MemRequest *mreq)
{ 
#ifdef MSHR_BWSTATS
  if(parallelMSHR)
    mshrBWHist.inc();
#endif
  //enforcing max ops/cycle for the specific bank
  doReadBankCB::scheduleAbs(nextBankSlot(mreq->getPAddr()), this, mreq);
}
void Cache::doReadBank(MemRequest *mreq)
{ 
  // enforcing max ops/cycle for the whole cache
  doReadCB::scheduleAbs(nextCacheSlot(), this, mreq);
}
void Cache::doRead(MemRequest *mreq)
{
  Line *l = getCacheBank(mreq->getPAddr())->readLine(mreq->getPAddr());

  if (l == 0) {
    if(isInWBuff(mreq->getPAddr())) {
      nForwarded.inc();
      mreq->goUp(fwdDelay);
      return;
    }
    readMissHandler(mreq);
    return;
  }

  readHit.inc();
  l->incReadAccesses();

  mreq->goUp(hitDelay);
}

//写

void Cache::write(MemRequest *mreq)
{ 
#ifdef MSHR_BWSTATS
  if(parallelMSHR)
    mshrBWHist.inc();
#endif
  doWriteBankCB::scheduleAbs(nextBankSlot(mreq->getPAddr()), this, mreq);
}
void Cache::doWriteBank(MemRequest *mreq)
{
  doWriteCB::scheduleAbs(nextCacheSlot(), this, mreq);
}

void Cache::doWrite(MemRequest *mreq)
{
  Line *l = getCacheBank(mreq->getPAddr())->writeLine(mreq->getPAddr());

  if (l == 0) {
    writeMissHandler(mreq);
    return;
  }

  writeHit.inc();
  l->makeDirty();

  mreq->goUp(hitDelay);
}
```
```Swift
//src/libmem/Cache.h
//该回调是一个对类Cache成员函数doRead()的调用，其参数仅有一个，且类型为MemRequest *。
  typedef CallbackMember1<Cache, MemRequest *, &Cache::doRead> 
    doReadCB;
  typedef CallbackMember1<Cache, MemRequest *, &Cache::doWrite> 
    doWriteCB;
```
在Cache对象中，access()函数是非常简单的，它发送读请求给read()函数和发送写请求给write()函数。每一个函数都使用一个回调以在下一个可获得的Cache周期中调用doRead()或doWrite()函数，这样可以模型化cache端口的竞争。如果读(写)在doRead()函数(doWrite()函数)是命中的，传进来的MemoryRequest对象参数的goUp()函数会被调用。这将会返回MemoryRequest给GProcessor。否则，一个miss处理函数将被调用，它会调用MemRequest对象的goDown()函数，并将MemRequest发送到层次结构中的下一级(如bus或L3 cache)。
```Java
//src/libmem/Cache.cpp
void Cache::readMissHandler(MemRequest *mreq)
{
  //......
  // Added a new MSHR entry, now send request to lower level
  readMiss.inc();
  sendMiss(mreq);
}

void WBCache::sendMiss(MemRequest *mreq)
{

  //......
  mreq->goDown(missDelay + (nextMSHRSlot(mreq->getPAddr())-globalClock), 
               lowerLevel[0]);
}
```

```Java
//src/libcore/MemRequest.h
  void goDown(TimeDelta_t lat, MemObj *newMemObj) {
    memStack.push(currentMemObj);
    clockStack.push(globalClock);
//......
    currentMemObj = newMemObj;
    accessCB.schedule(lat);
  }
```
##### 4.4 注意点
```Java
class MemObj {
public:
  typedef std::vector<MemObj*> LevelType;
private:
  bool highest;

protected:
  
  uint32_t nUpperCaches;
  LevelType upperLevel;
  LevelType lowerLevel;

  const char *descrSection;
  const char *symbolicName;

  void addLowerLevel(MemObj *obj) { 
    I( obj );
    lowerLevel.push_back(obj);
    obj->addUpperLevel(this);
  }

  void addUpperLevel(MemObj *obj) { 
    upperLevel.push_back(obj);
  }

  void invUpperLevel(PAddr addr, ushort size, MemObj *oc) {

    I(oc);
    
	 for(uint32_t i=0; i<upperLevel.size(); i++)
      upperLevel[i]->invalidate(addr, size, oc);    
  }
public:
//.....
  void computenUpperCaches();

  //This assumes single entry point for object, which I do not like,
  //but it is still something that is worthwhile.
  virtual Time_t getNextFreeCycle() const = 0;

  virtual void access(MemRequest *mreq) = 0;
  virtual void returnAccess(MemRequest *mreq) = 0;

  virtual void invalidate(PAddr addr, ushort size, MemObj *oc) = 0;
  virtual void doInvalidate(PAddr addr, ushort size) { I(0); }

  typedef CallbackMember2<MemObj, PAddr, ushort,
                         &MemObj::doInvalidate> doInvalidateCB;

  // When the buffers in the cache are full and it does not accept more requests
  virtual bool canAcceptStore(PAddr addr) = 0;
  virtual bool canAcceptLoad(PAddr addr) { return true; }

  // Print stats
  virtual void dump() const;
};
```

* 所有的Cache类和Bus类都是继承自MemObj类。MemObj类定义了一个公有的接口，如access()、returnAccess()和其他一些不太重要的函数。
* 当访问从一个较低层的cache返回到一个较高层的cache时，goUp()函数会调用returnAccess()函数。
* 公有接口允许完整的可配置cache层次结构。

