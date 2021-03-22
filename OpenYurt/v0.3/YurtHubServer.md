# YurtHubServer源码分析

YurtHubServer是执行ListenAndServe的主体，其主要结构和功能是YurtReverseProxy，即边缘节点的反向代理。YurtHubServer将YurtReverseProxy挂载在几乎所有路径（以`/`为前缀的路径，但也有一些路径另作他用）上，当HTTP请求的URL路径满足条件就会交由YurtReverseProxy处理，但在此之前会对request通过context添加一些额外的信息。



## 挂载的路径和功能

YurtHubServer挂载的路径，方法和handler的对应关系：

| Path                               | Method    | Handler       |
| ---------------------------------- | --------- | ------------- |
| `/v1/token`                        | POST, GET | updateToken   |
| `/v1/healthz`                      | GET       | healthz       |
| `/debug/pprof` and `/debug/pprof/` | ALL       | pprof.Index   |
| `/debug/pprof/profile`             | ALL       | pprof.Profile |
| `/debug/pprof/symbol`              | ALL       | pprof.Symbol  |
| `/debug/pprof/trace`               | ALL       | pprof.Trace   |
| Others                             | ALL       | proxyHandler  |

其中：

updateToken是用来更新节点的证书。

healthz是直接返回StatusOK，用于其它节点对本节点的健康检查。

pprof相关的handler是用来提供profile服务。

proxyHandler提供其主要的反向代理功能。



## proxyHandler功能

YurtHubServer在接收到HTTP request后，在通过YurtReverseProxy转发前，会先对request进行处理。流程如下：

1. 检查是否超过服务器request上限：

   当pending的request数量超过设置的阈值，拒绝请求。

2. 提取requestInfo：

   从request中提取具体信息requestInfo，其中包括verb（例如Watch, List），是否是resource请求等信息，并将requestInfo以键值对`requestInfoKey:requestInfo`的形式通过context附加在request上。

3. 附加ClientComponent信息：

   如果是resource请求，从`request.Header`中提取出`User-Agent`对应的值，以键值对`ProxyClientComponent:component`的形式通过context附加在request上。（component信息表示发送请求的是哪个部件，如kubelet，kube-proxy等）

4. 是否设置Timeout：

   如果`verb == Watch`，从request中提取Timeout（如果有的话）参数，以context with deadline的形式附加在request上。

5. 是否需要缓存：

   根据request.Header中`Edge-Cache`键对应的值`needToCache`（bool），以键值对`ProxyReqCanCache:needToCache`的形式通过context	附加在request上。

6. 附加ContentType信息：

   将request.Header中`Accept`键对应的值`contentType`，以键值对`ProxyReqContentType:contentType`的形式通过context附加在request上。

7. 跟踪response状态码

在这之后才会将request转交给YurtReverseProxy。



其构造handler的函数源码如下：

```go
func (p *yurtReverseProxy) buildHandlerChain(handler http.Handler) http.Handler {
	handler = util.WithRequestTrace(handler)
	handler = util.WithRequestContentType(handler)
	handler = util.WithCacheHeaderCheck(handler)
	handler = util.WithRequestTimeout(handler)
	handler = util.WithRequestClientComponent(handler)
	handler = filters.WithRequestInfo(handler, p.resolver)
	handler = util.WithMaxInFlightLimit(handler, p.maxRequestsInFlight)
	return handler
}
```

这里值得注意的是其执行顺序，当接收到request后，其通过的顺序应该是由下而上的。其中`With*`函数大体都是如下模式：

```go
func With*(handler http.handler, other-parameters) http.Handler{
	return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
		// some statements
        handler.ServeHTTP(w,req)
	})
}
```

因此在执行完`handler = util.WithRequestTrace(handler)`后，handler变为`with*`函数的返回值函数，但原先的handler并没有消失，而是在返回值函数中以`handler.ServeHTTP(w,req)`的形式存在。因此`with*`函数相当于在handler执行逻辑上增加了一些执行语句（some statements），对request处理时从最上面的语句开始执行，即从最后的`with*`函数逻辑开始向上执行。



## What's Next

[YurtReverseProxy](./ReverseProxy.md)