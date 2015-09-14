## What is Asynchronous Service? ##

> Asynchronous Service is service that won't block the current thread when it is executed. Guzz uses JDK5+'s concurrent package to provide this kind of service. When you execute a asynchronous service, the service is executing in a thread pool, and return back to the main thread until it is finished. In the meantime, the main thread can do anything else to achieve paralleling computing.

> Asynchronous Service in guzz can be used to improve a existing library, or to write a new asynchronous one.

## Write a Asynchronous Service. ##

> Most of the time, asynchronous tasks are remote call tasks. To call a remote resource, we usually use a user-defined protocol based on Socket, or use a RPC protocol directly.

### user-defined protocol ###

> To write a Asynchronous Service, you are suggested to inherit from org.guzz.service.AbstractRemoteService`<T>`, the T is the data type of the Service resulted. In your implementation, write you logic as usual until you need to perform a remote call. Then you call:

```
public FutureResult<T> sumbitTask(FutureDataFetcher<T> fetcher)
```

> This method returns a FutureResult. FutureResult is a asynchronous interface to your result, call its methods when you need to use the result later.

> The interface of FutureDataFetcher is:

```
package org.guzz.service;

public interface FutureDataFetcher<T> extends java.util.concurrent.Callable<T>{		
	public T getDefaultData() ;
}
```

FutureDataFetcher has two methods:

> public T call() throws Exception ; The method to perform the actual operation. This method can be synchronous, such as connecting to a memcached and performing storing.

> public T getDefaultData() ; The default value to return when the task is canceled or failed, and you are reading result ignoring any exceptions.

AbstractRemoteService use JDK5's ExecutorService to perform the asynchronous call. Injected the ExecutorService with service "org.guzz.service.core.impl.JDK5ExecutorServiceImpl", eg:

```
<service name="executorService" configName="jdk5ExecutorService" class="org.guzz.service.core.impl.JDK5ExecutorServiceImpl" />	

<service name="yourService" dependsOn="executorService" configName="yourServiceConf" class="your service's class name" />	
```

And tune the ExecutorService's thread pool with the below parameters in the config file:

```
[jdk5ExecutorService]
#min threads
corePoolSize=5

#max threads
maxPoolSize=50

#max idle time(million seconds). The idle thread will be killed, until the total number of threads is dropped to corePoolSize.
keepAliveMilSeconds=60000

#max queue size if there is no idle thread.
queueSize=2048
```

> The sample config shows the default parameters for the thread pool, it is optional.

**TIPS**: AbstractRemoteService is just a abstract service providing some asynchronous methods. The asynchronous calls can be a "remote" one, or a local JVM one, AbstractRemoteService doesn't rely on network.

### RPC Service ###

> RPC is the short name of Remote Procedure Call. It uses a local stub interface to call a remote service, and hides the complexity of the network protocols. RPC has many protocols and providers, such as RMI, hessian, phprpc, and so on.

> Guzz has done some preparation for using RPC in service. You service can inherit from org.guzz.service.AbstractRPCService`<T>` to benefit from it.

> AbstractRPCService is derived from AbstractRemoteService, so the FutureResult and thread pool parameters are still valid here.

> AbstractRPCService use a standard way to initialize the RPC proxy on startup, so you can change the RPC provider in the future when needed without changing any codes. The RPC provider is declared by the rpc.protocol parameter in the config file. The rpc.protocal value is the proxy's name or the full class name of the proxy implementation(a class implements org.guzz.service.remote.RemoteRPCProxy). For example:

```
#set the RPC provider to phprpc
rpc.protocol=phprpc

#this parameter is a parameter of the phprpc provider itself.
rpc.serviceURL=http://services.guzz.org/service/IPService
```

> In the implementation of a service, whether asynchronous or not, is decided by the developers. If you don't need asynchronous, call and return the result directly; If you need it, call the parent's method:
```
public FutureResult<T> sumbitTask(FutureDataFetcher<T> fetcher)
```

and return the FutureResult.

### RPC providers guzz supported now: ###

| RPC Provider | rpc.protocol | Parameters | More |
|:-------------|:-------------|:-----------|:-----|
| hessian      | hessian        | rpc.url the service url<br>Any properties can be called by setXXX in com.caucho.hessian.client.HessianProxyFactory(the parameter name is:rpc.xXX).<br>eg: rpc.user, rpc.password, rpc.connectionFactoryName, rpc.debug, rpc.overloadEnabled, rpc.chunkedPost, rpc.readTimeout, rpc.hessian2Request. <table><thead><th> <a href='http://hessian.caucho.com/'>http://hessian.caucho.com/</a> </th></thead><tbody>
<tr><td> burlap       </td><td> burlap       </td><td> rpc.url the service url<br>Any properties can be called by setXXX in com.caucho.burlap.client.BurlapProxyFactory(the parameter name is:rpc.xXX).<br>eg: rpc.user, rpc.password, rpc.overloadEnabled. </td><td> <a href='http://hessian.caucho.com/doc/burlap.xtp'>http://hessian.caucho.com/doc/burlap.xtp</a> </td></tr>
<tr><td> PHPRPC       </td><td> phprpc         </td><td> rpc.serviceURL the service url<br>Any properties can be called by setXXX in org.phprpc.PHPRPC_Client(the parameter name is:rpc.xXX). </td><td> <a href='http://www.phprpc.org/'>http://www.phprpc.org/</a> </td></tr>