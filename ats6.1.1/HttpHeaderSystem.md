# Traffic Server HTTP Header System
## No Null-Terminated Strings
反正在使用traffic server 中的字符串的时候不能假定有正常结束符。所以正常的c语言的字符串处理函数类似[str*()] 这样的要非常小心。一般要结合长度进行使用。如果需要有正常结束符的字符串，建议使用TSstrndup 函数进行拷贝。从头信息里面取出来的字符串可能是NULL的，表示头部信息不存在。  

``` c++  
    char *host_string;
    int host_length;
    host_string = TSUrlHostGet (bufp, url_loc, &host_length);
    for (i = 0; i < nsites; i++) {
    //指定长度进行比较
    if (strncmp (host_string, sites[i], host_length) == 0) {
    // ...
    }
```  
## Duplicate MIME Fields Are Not Coalesced
这个部分的意思大概就是 MIME Fields 可能有重复。`TSMimeHdrField` 系列函数   

## Release Marshal Buffer Handles
使用HTTP头部处理相关函数获取一个对象句柄，`TSMLoc `或者 `char *`。最后投药通过主动释放这些句柄所持有的资源。`TSUrlStringGet` 比较特殊需要使用`TSfree`进行资源的释放。
TSHandleMLocRelease 函数有三个参数，意义分别如下：
* the marshal buffer containing the data
* the location of the parent object
* the location of the object to be released 
``` c++
/*
TSReturnCode
TSHandleMLocRelease(TSMBuffer bufp, TSMLoc parent, TSMLoc mloc)
*/
url_loc = TSHttpHdrUrlGet (bufp, hdr_loc);
host_string = TSUrlHostGet (bufp, url_loc, &host_length);
//then your plugin would have to call:
TSHandleMLocRelease (bufp, hdr_loc, url_loc);
```
像那些没有父`TSMLoc` 的，那么就传入`TS_NULL_MLOC` 这种参数就可以了。
``` c++
TSHttpTxnClientReqGet (txnp, &bufp, &hdr_loc);
then you must release hdr_loc with:

TSHandleMLocRelease (bufp, TS_NULL_MLOC, hdr_loc);
```
必须使用`TS_NULL_MLOC`  去释放由`TSHttpTxn*Get` 生成的 `TSMLoc` 

``` c++
TSHttpTxnServerRespGet( txnp, &resp_bufp, &resp_hdr_loc );
new_field_loc = TSMimeHdrFieldCreate (resp_bufp, resp_hdr_loc);
TSHandleMLocRelease ( resp_bufp, resp_hdr_loc, new_field_loc);
TSHandleMLocRelease ( resp_bufp, TS_NULL_MLOC, resp_hdr_loc);
```

>注意 一定要在`TSHttpTxnReenable`之前调用 TSHandleMLocRelease 

***
示例：
* TSReturnCode TSHttpTxnServerRespGet(TSHttpTxn txnp, TSMBuffer * bufp, TSMLoc * offset)  
* TSMLoc TSMimeHdrFieldFind(TSMBuffer bufp, TSMLoc hdr, const char * name, int length)  
* const char * TSMimeHdrFieldValueStringGet(TSMBuffer bufp, TSMLoc hdr, TSMLoc field, int idx, int * value_len_ptr)
* TSHandleMLocRelease  

```c
#include <string.h>
#include <ts/ts.h>

int
get_content_type(TSHttpTxn txnp, char* buf, size_t buf_size)
{
  TSMBuffer bufp;
  TSMLoc hdrs;
  TSMLoc ctype_field;
  int len = -1;

  if (TS_SUCCESS == TSHttpTxnServerRespGet(txnp, &bufp, &hdrs)) {
    ctype_field = TSMimeHdrFieldFind(bufp, hdrs, TS_MIME_FIELD_CONTENT_TYPE, TS_MIME_LEN_CONTENT_TYPE);

    if (TS_NULL_MLOC != ctype_field) {
      const char* str = TSMimeHdrFieldValueStringGet(bufp, hdrs, ctype_field, -1, &len);

      if (len > buf_size)
        len = buf_size;
      memcpy(buf, str, len);
      TSHandleMLocRelease(bufp, hdrs, ctype_field);
    }
    TSHandleMLocRelease(bufp, TS_NULL_MLOC, hdrs);
  }

  return len;
}
```
