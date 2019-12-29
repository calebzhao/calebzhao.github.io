---
title: 2、spring cloud ribbon源码分析
date: 2019-12-29 16:48:27
tags:
- spring cloud
- ribbon
categories: spring cloud
---

、RestTemplate调用原理

1、调用restTemlate.get()

2、进入RestTemplate的拦截器
```java
public class InterceptingClientHttpRequest{

  private class InterceptingRequestExecution implements ClientHttpRequestExecution

    @Override
     public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
	   if (this.iterator.hasNext()) {
		  ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
		  return nextInterceptor.intercept(request, body, this);
	   }
	   else {
		 ...省略
	   }
     }

}
```


3、 进入LoadBalancerInterceptor 拦截器，其中LoadBalancerClient的实现一般为RibbonLoadBalancerClient
```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
				final ClientHttpRequestExecution execution) throws IOException {

			final URI originalUri = request.getURI();
			String serviceName = originalUri.getHost();
			Assert.state(serviceName != null,"Request URI does not contain a valid hostname: " + originalUri);
			return this.loadBalancer.execute(serviceName, this.requestFactory.createRequest(request, body, execution));

		}
	}
}
```
4、进入ribbon的客户端负载均衡实现，RibbonLoadBalancerClient.execute()方法
```java
public class RibbonLoadBalancerClient implements LoadBalancerClient {

	public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint) throws IOException {
		 // 获取所有实例列表
		 ILoadBalancer loadBalancer = this.getLoadBalancer(serviceId);
		// 按照负载均衡算法选择1个实例
		 Server server = this.getServer(loadBalancer, hint);
		if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		} 
		else {
		 	RibbonLoadBalancerClient.RibbonServer ribbonServer = new RibbonLoadBalancerClient.RibbonServer(serviceId, server, 	this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));
			# 向选择的实例真正发送请求
			return this.execute(serviceId, (ServiceInstance)ribbonServer, (LoadBalancerRequest)request);
		}
	}

}
```