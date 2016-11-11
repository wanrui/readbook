# continuation

```c++
// 通过Continuation 构造Event。然后可将生成的Event 丢入ethread 中执行（并非执行，而是最后丢入队列中）
TS_INLINE Event *
Event::init(Continuation * c, ink_hrtime atimeout_at, ink_hrtime aperiod)
{
  continuation = c;
  timeout_at = atimeout_at;
  period = aperiod;
  immediate = !period && !atimeout_at;
  cancelled = false;
  return this;
}
```

```c++
// 通过EventProcessor 执行Continuation 。其中会有Continuation转换为Event，并且传入参数中有指定线程类型。
TS_INLINE Event *
EventProcessor::schedule_in(Continuation * cont, ink_hrtime t, EventType et, int callback_event, void *cookie)
{
  Event *e = eventAllocator.alloc();

  ink_assert(et < MAX_EVENT_TYPES);
  e->callback_event = callback_event;
  e->cookie = cookie;
  return schedule(e->init(cont, ink_get_based_hrtime() + t, 0), et);
}

//EventProcessor 将Continuation 转化为Event 后丢入EventQueueExternal 队列中。
TS_INLINE Event *
EventProcessor::schedule(Event * e, EventType etype, bool fast_signal)
{
  ink_assert(etype < MAX_EVENT_TYPES);
  e->ethread = assign_thread(etype);
  if (e->continuation->mutex)
    e->mutex = e->continuation->mutex;
  else
    e->mutex = e->continuation->mutex = e->ethread->mutex;
  e->ethread->EventQueueExternal.enqueue(e, fast_signal);
  return e;
}
```

```c++
//
// General case network connection accept code
//
//eventsystem异步执行某个函数的方法 先设置将Continuation 的handler。把需要执行的代码放入handler 中，然后把Continuation丢入eventProcessor中执行。
int
net_accept(NetAccept * na, void *ep, bool blockable)
{
  Event *e = (Event *) ep;
  int res = 0;
  int count = 0;
  int loop = accept_till_done;
  UnixNetVConnection *vc = NULL;

  if (!blockable)
    if (!MUTEX_TAKE_TRY_LOCK_FOR(na->action_->mutex, e->ethread, na->action_->continuation))
      return 0;
  //do-while for accepting all the connections
  //added by YTS Team, yamsat
  do {
    vc = (UnixNetVConnection *) na->alloc_cache;
    if (!vc) {
      vc = na->allocateThread(e->ethread);
      NET_SUM_GLOBAL_DYN_STAT(net_connections_currently_open_stat, 1);
      vc->id = net_next_connection_number();
      na->alloc_cache = vc;
    }
    if ((res = na->server.accept(&vc->con)) < 0) {
      if (res == -EAGAIN || res == -ECONNABORTED || res == -EPIPE)
        goto Ldone;
      if (na->server.fd != NO_FD && !na->action_->cancelled) {
        if (!blockable)
          na->action_->continuation->handleEvent(EVENT_ERROR, (void *)(intptr_t)res);
        else {
          MUTEX_LOCK(lock, na->action_->mutex, e->ethread);
          na->action_->continuation->handleEvent(EVENT_ERROR, (void *)(intptr_t)res);
        }
      }
      count = res;
      goto Ldone;
    }
    count++;
    na->alloc_cache = NULL;

    vc->submit_time = ink_get_hrtime();
    ats_ip_copy(&vc->server_addr, &vc->con.addr);
    vc->mutex = new_ProxyMutex();
    vc->action_ = *na->action_;
    vc->set_is_transparent(na->server.f_inbound_transparent);
    vc->closed  = 0;
    SET_CONTINUATION_HANDLER(vc, (NetVConnHandler) & UnixNetVConnection::acceptEvent);

    if (e->ethread->is_event_type(na->etype))
      vc->handleEvent(EVENT_NONE, e);
    else
      eventProcessor.schedule_imm(vc, na->etype);
  } while (loop);

Ldone:
  if (!blockable)
    MUTEX_UNTAKE_LOCK(na->action_->mutex, e->ethread);
  return count;
}
```
高并发产生的c10k问题的解决。在多线程模型下，c10k的i/o策略包含有这么两种："one OS-level thread handles many clients; each client is controlled by a state machine"以及"one OS-level thread handles many clients; each client is controlled by a continuation"。一个state machine由一组有序的continuation构成，一个continuation完成后，thread会执行下一个continuation。这两种策略的区别在于，对于第一种策略，整个state machine的所有continuation都是由一个thread来完成的，而对于第二种策略，由于thread处理的单位是一个continuation，所以一个state machine可能会由几个thread完成(个人看法)。
trafficserver采用的是第二种i/o策略，同时，采用event-based的机制来调度线程完成每个continuation。在trafficserver中，类Continuation是所有不同类型的continuation的父类，它提供一个回调函数指针，执行一个continuation就是执行这个回调函数。类Event代表的是一个event，它封装了一个Continuation，类Ethread代表的是一个线程，线程的调度是由类Processor的子类EventProcessor来完。trafficserver创建两种类型的线程，regular与dedicated类型。一个dedicated类型的线程，它在执行完某个continuation后，就消亡了。regular类型的线程，在trafficserver启动后，就一直存在着，它的线程执行函数是一个死循环。regular线程维护了一个event pool，当event pool不为空时，线程就从event pool中取出event，同时执行event封装的continuation。当需要调度某个线程执行某个continuation时，通过EventProcessor将一个continuation封装成一个event并将其加入到线程的event pool中。
定时器的支持。trafficserver将event分为以下几种类型：立即执行的event，定时执行的event，定期执行的event以及随时执行的event。一个event包含一个timeout时间，一个period时间，timeout是一个非负数，period没有限制。
1) 当timeout等于0，该event是一个立即event。  
2) 当period等于0，该event是一个定时event，timeout时间到期后event就会执行。  
3) 当period大于0，该event是一个定期执行的event，每隔period就会执行一次。  
4) 如果period小于0，event是一个随时执行的event，当线程执行完其他类型的event后，就会执行这一类event。目前ts只有一种该类型的event，就是epoll，这样在工作线程没有其他任务时就立即执行epoll_wait。  
为了支持定时器，trafficserver将线程的event pool细分为一个外部队列与一个内部队列，以及一个优先级队列。当调度某个线程执行某个event时，EventProcessor将该event加入到线程的外部队列中。线程从外部队列中取出event加入到自己的内部队列处理，同时，将定时或者定期event加入到优先级队列中。优先级队列是一个粗粒度的管理定时event执行的队列，它根据时间段的大小维护了不同的桶，不同时间后执行的event加入到不同的桶中。
eventsystem核心代码与数据结构

```c
EThread::execute                      //对应线程执行函数，对于regular类型的线程，是一个死循环
//这一组函数，选择类型与EventType参数值相同的那组线程，从该组线程中以round robin方式选出一个线程，
将event插入到该线程的外部队列中
EventProcessor::schedule_imm          //立即执行event
EventProcessor::schedule_at           //将来某个确定的时间点执行event
EventProcessor::schedule_in           //一段时间以后执行event
EventProcessor::schedule_every        //定期或随时event，根据参数给出的时间是正值还是负值
EThread::schedule_*                   //这一类函数大致与EventProcessor提供的相同
EThread::schedule_*_local             //这一类函数与上面的唯一不同是，将event直接加入内部队列中
Event::schedule_*                     //这一类函数也是将event直接加入到对应线程的内部队列中

//这两个函数其实是专为网络模块设计的，一个线程可能一直阻塞在epoll_wait上，通过引入一个pipe或者eventfd，当调度一个线程执行某个event时，异步通知该线程从epoll_wait解放出来。
EThread::schedule_imm_signal          
EventProcessor::schedule_imm_signal
```

上传下载的数据
UnixNetVConnection 设置NetVConnHandler 为UnixNetVConnection::acceptEvent，然后使用eventProcessor.schedule_imm立即执行。 

```c++
class Example : public Continuation
{
public:
    Example();
    ~Example();
    int main_event(int event, Event *e);   //将执行的代码写入该函数
};
```

```c++
Example *e = new Example();                         //创建一个对象
SET_CONTINUATION_HANDLER(e, &Example::main_event); //设置异步执行的函数
eventProcessor.schedule_imm(e, ET_CALL);           //异步执行，eventProcessor是ts提供的全局变量
```