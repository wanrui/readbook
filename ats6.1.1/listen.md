# 服务器listen过程

proxy\Main.cc

```
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
