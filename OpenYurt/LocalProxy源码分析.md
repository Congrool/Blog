# LocalProxy源码分析



## LocalProxy的主要功能

在远端节点非健康状态下，接受来自YurtReverseProxy的request，并根据本地缓存和request的内容进行处理。目前只支持ResourceRequest。处理内容主要是根据ResourceRequest中对资源的操作类型（verb）来更新本地缓存。



## LocalProxy的结构

LocalProxy内有：

1. CacheManager：用来管理本地缓存
2. IsHealthy：是传入的LoadBalancer功能，用来判断远端节点是否健康

结构体如下：

```go
// LocalProxy is responsible for handling requests when remote servers are unhealthy
type LocalProxy struct {
	cacheMgr  manager.CacheManager
	isHealthy IsHealthy
}
```



## LocalProxy的处理流程

1. 过滤非ResourceRequest，并返回BadRequest Response

2. 根据操作类型分别进行处理：

   - **watch：**

     在Response.Header中设置`Transfer-Encoding:chuncked`，以长连接处理watch请求。然而目前LocalProxy并不支持本地处理watch，因此直到Timeout都在周期性探测远端节点是否健康，如果感知到远端节点恢复健康则直接返回nil，下次watch将由LoadBalancer处理。

   - **create：**

     如果请求的resource是event，则调用`CacheManager.CacheResponse`将请求的内容缓存到本地。并总会返回一个response，header和body与request相同，状态码为201(StatusCreated)。

   - **delete || deletecollection：**

     LocalProxy不支持对resource的delete请求，因此返回的Response的Reason为"Forbidden"，状态码为200(StatusOK)。

   - **list || get || update || patch：**

     根据request中的信息生成key，调用`CacheManager.QueryCache`根据key找到缓存中的runtime.object(s)，将其写入response并返回。

服务功能的伪代码如下：

```
if reqestInfo != nil and requestInfo.IsResourceRequest{
	switch requestInfo.Verb{
	case "watch":
		localWatch(request)
	case "create"：
		localPost(request)
	case "delete" or "deletecollection":
		localDelete(request)
	default:
		localReqCache(request)
	}
}
else{
	resp.Write(errors.BadRequest)
}
```





## What's next

[CacheManager源码分析](./CacheManager源码分析.md)