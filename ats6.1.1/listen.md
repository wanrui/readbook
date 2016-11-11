# 服务器listen过程

proxy\Main.cc

```c
      // Delay only if config value set and flag value is zero
      // (-1 => cache already initialized)
      if (delay_p && ink_atomic_cas(&delay_listen_for_cache_p, 0, 1)) {
        Debug("http_listen", "Delaying listen, waiting for cache initialization");
      } else {
        start_HttpProxyServer(); // PORTS_READY_HOOK called from in here
      }
```


``` c++
void init_HttpProxyServer(int n_accept_threads) 

static void MakeHttpProxyAcceptor(HttpProxyAcceptor &acceptor, HttpProxyPort &port, unsigned nthreads)
```  

```c
start_HttpProxyServer()---> netProcessor.main_accept(acceptor._accept, port.m_fd, acceptor._net_opt))
Action * UnixNetProcessor::accept_internal(Continuation *cont, int fd, AcceptOptions const &opt)
  NetAccept *na = createNetAccept();
   na->server.fd = fd;
 na->accept_fn = net_accept; // All callers used this.
 na->action_ = new NetAcceptAction();
 na->do_listen(BLOCKING, opt.f_inbound_transparent)
  a->init_accept_loop(thr_name); 
```  


```c
void
NetAccept::init_accept(EThread *t, bool isTransparent)
{
  if (!t)
    t = eventProcessor.assign_thread(etype);

  if (!action_->continuation->mutex) {
    action_->continuation->mutex = t->mutex;
    action_->mutex = t->mutex;
  }
  if (do_listen(NON_BLOCKING, isTransparent))
    return;
  SET_HANDLER((NetAcceptHandler)&NetAccept::acceptEvent);
  period = -HRTIME_MSECONDS(net_accept_period);
  t->schedule_every(this, period, etype);
}


int
NetAccept::acceptEvent(int event, void *ep)
{
  (void)event;
  Event *e = (Event *)ep;
  // PollDescriptor *pd = get_PollDescriptor(e->ethread);
  ProxyMutex *m = 0;

  if (action_->mutex)
    m = action_->mutex;
  else
    m = mutex;
  MUTEX_TRY_LOCK(lock, m, e->ethread);
  if (lock.is_locked()) {
    if (action_->cancelled) {
      e->cancel();
      NET_DECREMENT_DYN_STAT(net_accepts_currently_open_stat);
      delete this;
      return EVENT_DONE;
    }

    // ink_assert(ifd < 0 || event == EVENT_INTERVAL || (pd->nfds > ifd && pd->pfd[ifd].fd == server.fd));
    // if (ifd < 0 || event == EVENT_INTERVAL || (pd->pfd[ifd].revents & (POLLIN | POLLERR | POLLHUP | POLLNVAL))) {
    // ink_assert(!"incomplete");
    int res;
    if ((res = accept_fn(this, e, false)) < 0) {
      NET_DECREMENT_DYN_STAT(net_accepts_currently_open_stat);
      /* INKqa11179 */
      Warning("Accept on port %d failed with error no %d", ats_ip_port_host_order(&server.addr), res);
      Warning("Traffic Server may be unable to accept more network"
              "connections on %d",
              ats_ip_port_host_order(&server.addr));
      e->cancel();
      delete this;
      return EVENT_DONE;
    }
    //}
  }
  return EVENT_CONT;
}
```