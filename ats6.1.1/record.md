#

## proxy.config.http.enabled  
Scope:	CONFIG  
Type:	INT  
Default:	0  
***
打开或关闭支持HTTP代理。这是很少使用,唯一的例外的情况是是如果你使用了protocol插件,并且希望它不支持HTTP请求

## proxy.config.http.wait_for_cache  
Scope:	CONFIG  
Type:	INT  
Default:	1  
***
接受入站连接,在通信服务器缓存是独立的业务。这个变量控制这些操作的相对时间和交通需要依赖于缓存,因为如果缓存服务器,然后接受入站连接应该推迟到缓存的有效性要求确定。将登录diags.log缓存的初始化失败。


* 0 ：入站连接和缓存的初始化是解耦的。无论缓存的初始化是否完成，连接都会被尽快接收。
* 1 ：缓存的初始化完成之前不接受入站连接。ts 会在缓存初始化完成后运行。 
* 2 ：
* 3 ：缓存的初始化完成之前不接受入站连接。ts 会在缓存初始化完成后运行。

sudo ./traffic_line -r proxy.process.net.connections_currently_open