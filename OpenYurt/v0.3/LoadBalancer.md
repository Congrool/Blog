# LoadBalancer源码分析



## LoadBalancer的主要功能

在APIServer处于健康状态时，LoadBalancer接收来自YurtReverseProxy的request，并转发到APIServer。LoadBalancer中记录了多个能处理request的APIServer的地址，在转发request时会采取一定的负载均衡策略。



## 结构

### LoadBalancer的结构

LoadBalancer内主要有APIServer代理列表（`[]*RemoteProxy`)和负载均衡策略（`loadBalancerAlgo`）。APIServer代理（RemoteProxy）代理一个能处理request的APIServer。LoadBalancer根据设置的负载均衡策略，挑选一个APIServer代理，将request通过APIServer代理转发到APIServer。

```go
type loadBalancer struct {
	backends    []*RemoteProxy
	algo        loadBalancerAlgo
	certManager interfaces.YurtCertificateManager
}
```

### RemoteProxy的结构

RemoteProxy内置有

- RemoteProxy：利用`net/http/httputil`中的ReverseProxy来完成代理的主要工作。
- CacheManager：利用缓存中的信息修改从APIServer返回的response
- HealthChecker，用来判断APIServer是否健康
- RemoteServer，用来记录APIServer的信息。

```go
// RemoteProxy is an reverse proxy for remote server
type RemoteProxy struct {
	checker      healthchecker.HealthChecker
	reverseProxy *httputil.ReverseProxy
	cacheMgr     cachemanager.CacheManager
	remoteServer *url.URL
	stopCh       <-chan struct{}
}
```



## LoadBalancer的工作流程

1. 根据一定的负载均衡策略选择一个健康的APIServer
2. 将request交APIServer代理处理

源码如下：

```go
func (lb *loadBalancer) ServeHTTP(rw http.ResponseWriter, req *http.Request){
    // 通过策略选择一个APIServer代理
	b := lb.algo.PickOne()
    
    // ...
	// omit code of error handling and logging
    // ...
    
    // 交由APIServer代理处理
	b.ServeHTTP(rw, req)
}
```

APIServer代理收到request后直接调用httputil中的RemoteProxy处理。

```go
func (rp *RemoteProxy) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
	rp.reverseProxy.ServeHTTP(rw, req)
}
```

除了默认的处理流程外，将收到response发回前可能还会对response的内容进行处理和修改。主要为：

1. 如果request的操作为"watch"，会在response.Header中添加`Transfer-Encoding:chunked`，使用长连接处理watch请求。
2. 如果request设置了缓存（根据request的context的`ProxyReqCanCache:needToCache`，或是使用其它条件判断），则会调用CacheManager将reponse的内容缓存到本地。



## 负载均衡策略

目前LoadBalancer中内置了两种负载均衡策略：Round-Robin和基于优先级的策略。一个LoadBalancer采用哪种负载均衡策略在创建时由`lbMode`参数指定：`rr`表示Round-Robin，`priority`表示基于优先级的调度策略。默认使用的Round-Robin。LoadBalancer统一通过策略的PickOne()接口来选出一个能够转发请求的APIServer。

**策略结构**

```go
type loadBalancerAlgo interface {
	PickOne() *RemoteProxy
	Name() string
}
```



**Roud-Robin策略原理**

如果没有健康的APIServer，则返回nil。否则通过不回退的轮询来找到第一个健康的APIServer。

轮询部分的源码：

```go
hasFound := false
selected := rr.next // next 是上次轮询的终点位置
// rr.backends是记录的所有APIServer的序列
for i := 0; i < len(rr.backends); i++ {
    selected = (rr.next + i) % len(rr.backends)
    if rr.backends[selected].IsHealthy() {
        hasFound = true
        break
    }
}

if hasFound {
    rr.next = (selected + 1) % len(rr.backends)
    return rr.backends[selected]
}
```



**基于优先级的策略**

使用基于优先级的策略时，需要手动按优先级将APIServer序列排序。策略本身的实现是从头开始的轮询。

实现源码：

```go
// prio.backends是记录的所有APIServer的序列
for i := 0; i < len(prio.backends); i++ {
    if prio.backends[i].IsHealthy() {
    	return prio.backends[i]
    }
}
```

