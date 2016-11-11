# appache traffic server plugin(ATS 6.1.1)
## Stats Over HTTP Plugin
文件名：stats_over_http.so 
类型（全局/remap）：全局插件  
功能：提供web 服务，把当前的record 统计信息以json 的格式反馈给终端（然而没什么卵用） 
原理：  
* TSHttpHookAdd; 捕获TS_HTTP_READ_REQUEST_HDR_HOOK 请求，
* 获取请求信息，判定是获取统计信息的请求，并通过接口 TSSkipRemappingSet(txnp, 1)绕过remap 
* 通过接口TSHttpTxnIntercept，拦截处理，把处理响应交由stats_dostuff函数；
* 通过函数stats_dostuff进行进行对客户端响应  
```c
static int
stats_dostuff(TSCont contp, TSEvent event, void *edata)
{
  stats_state *my_state = TSContDataGet(contp);
  if (event == TS_EVENT_NET_ACCEPT) {
    my_state->net_vc = (TSVConn)edata;
    stats_process_accept(contp, my_state);
  } else if (edata == my_state->read_vio) {
    stats_process_read(contp, event, my_state);
  } else if (edata == my_state->write_vio) {
    stats_process_write(contp, event, my_state);
  } else {
    TSReleaseAssert(!"Unexpected Event");
  }
  return 0;
}
```  
* 通过接口TSRecordDump读取ATS的record信息
```c 
TSRecordDump((TSRecordType)(TS_RECORDTYPE_PLUGIN | TS_RECORDTYPE_NODE | TS_RECORDTYPE_PROCESS), 
            json_out_stat,
            my_state)
```
* 追加到响应报文中

配置说明： 
 stats_over_http.so 后面可以更一个识别码。 用来防止别人乱访问的。
配置示例：  
stats_over_http.so 
例如ats 的ip 为192.168.1.159 端口为8080，配置上这个插件后，我们可以通过 http://192.168.1.159:8080/_stats
***
## TCPInfo Plugin  
文件名：tcpinfo.so  
类型（全局/remap）：全局插件  
功能：请求过来链接的TCP信息输出来。包含rrt、cwnd等相关信息。  
原理：  
* 通过接口 TSHttpSsnClientFdGet(ssnp, &fd) 获取客户端到本机的的FD；
* 通过接口 getsockopt(fd, IPPROTO_TCP, TCP_INFO, &info, &tcp_info_len)  获取TCP对应的信息；
* 通过 random = rand() % 1000;random < config->sample 控制采样率；
* 通过TSHttpHookAdd 捕获不同阶段的事件，触发TCP信息统计，记录到对应的日志文件。

配置说明： 
* --log-file 指定log 生成目录  ；
* --log-level 指定 log 生成等级，1，就是HTTP层 2；
* --hooks=NAMELIST ，制定需要捕获的事件，事件为指定的类型 ，中间使用逗号‘,’隔开；
   * send_resp_hdr	The server begins sending an HTTP response.
   * ssn_close	The TCP connection closes.
   * ssn_start	A new TCP connection is accepted.
   * txn_close	A HTTP transaction is completed.
   * txn_start	A HTTP transaction is initiated.  
* --sample-rate=COUNT  设置采样率。1000为满分 
* --log-level=LEVEL 有两种生成模式，生成的内容也不大相同。  
```c
      if (config->log_level == 2) {
#if !defined(freebsd) || defined(__GLIBC__)
    ret = TSTextLogObjectWrite(config->log, "%s %s %s %u %u %u %u %u %u %u %u %u %u %u %u", event_name, client_str, server_str,
                               info.tcpi_rtt, info.tcpi_rttvar, info.tcpi_last_data_sent, info.tcpi_last_data_recv,
                               info.tcpi_snd_cwnd, info.tcpi_snd_ssthresh, info.tcpi_rcv_ssthresh, info.tcpi_unacked,
                               info.tcpi_sacked, info.tcpi_lost, info.tcpi_retrans, info.tcpi_fackets);
#else
    ret = TSTextLogObjectWrite(config->log, "%s %s %s %u %u %u %u %u %u %u %u %u %u %u %u", event_name, client_str, server_str,
                               info.tcpi_rtt, info.tcpi_rttvar, info.__tcpi_last_data_sent, info.tcpi_last_data_recv,
                               info.tcpi_snd_cwnd, info.tcpi_snd_ssthresh, info.__tcpi_rcv_ssthresh, info.__tcpi_unacked,
                               info.__tcpi_sacked, info.__tcpi_lost, info.__tcpi_retrans, info.__tcpi_fackets);
#endif
  } else {
    ret = TSTextLogObjectWrite(config->log, "%s %s %s %u", event_name, client_str, server_str, info.tcpi_rtt);
  } 
 ```
* --log-file=NAME 指定文件生成目录。

配置示例： 
tcpinfo.so --log-file=tcp-metrics --log-level=2 --hooks=ssn_start,txn_start,send_resp_hdr,ssn_close,txn_close  


