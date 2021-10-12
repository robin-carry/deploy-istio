# Lab 1 :: Run Envoy Proxy

In this lab, we dig into one of the foundational pieces of Istio. The "data plane", or the service proxy that lives with each service/application instance is on the request path on both origination of a service call as well as usually on the destination side of the service call. 

The service proxy that Istio uses is [Envoy Proxy](https://www.envoyproxy.io). Envoy is an incredibly powerful and well-suited proxy for this use case. 

It's impossible to overstate how important Envoy is to Istio, which is why we start the labs with it.

## Prerequisites

You will need access to a Kubernetes cluster. If you're doing this via the Solo.io Workshop format, you should have everything ready to go.

Verify you're in the correct folder for this lab: `/home/solo/workshops/istio-day2/1-deploy-istio/`. 

## Set up supporting services

We use a simple `httpbin` service as well as `sleep` app to exercise the basic functionality of Envoy is this lab.

```bash
kubectl apply -f labs/01/httpbin.yaml
kubectl apply -f labs/01/sleep.yaml
```

To verify we have things installed correctly, let's try running it:

<!--bash
until kubectl exec deploy/sleep -- curl httpbin:8000/headers -sSL -m 1 >/dev/null 2>&1
do
  sleep 1
done
-->
```bash
kubectl exec deploy/sleep -- curl httpbin:8000/headers
```

{% hint style="info" %}
Note, it may take a few moments for the `sleep` pod to come up, so you may need to retry the previous command if it fails
{% endhint %}


We should see httpbin output that looks similar to this:

```
{
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin:8000", 
    "User-Agent": "curl/7.69.1"
  }
}
```


## Review Envoy proxy config

Envoy can be configured completely by loading a YAML/JSON file or in part with a dynamic API. The dynamic API is a big reason why microservice networking frameworks like Istio use Envoy, but we start by first understanding the configuration from a basic level. 

We will  use the file configuration format. Take a look at as simple configuration file:

```
cat labs/01/envoy-conf.yaml
```

```
admin:
  accessLogPath: /dev/stdout
  address:
    socketAddress:
      address: 0.0.0.0
      portValue: 15000
staticResources:
  listeners:
  - name: httpbin-listener
    address:
      socketAddress:
        address: 0.0.0.0
        portValue: 15001
    filterChains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typedConfig:
          '@type': type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          httpFilters:
          - name: envoy.filters.http.router
          routeConfig:
            name: simple_httpbin_route
            virtualHosts:
            - domains:
              - '*'
              name: httpbin_host
              routes:
              - match:
                  prefix: /
                route:
                  cluster: httpbin_service
          statPrefix: httpbin
  clusters:
  - connectTimeout: 5s
    loadAssignment:
      clusterName: httpbin_service
      endpoints:
      - lbEndpoints:
        - endpoint:
            address:
              socketAddress:
                address: httpbin
                portValue: 8000
    name: httpbin_service
    respectDnsTtl: true
    dnsLookupFamily: V4_ONLY
    type: STRICT_DNS
    upstreamConnectionOptions:
      tcpKeepalive: {}
```

In this configuration, we see three main sections:

* [Admin] - Setting up the administration API for the proxy
* [Listeners] - Declaration of ports to open on the proxy and listen for incoming connections
* [Clusters] - Backend service to which we can route traffic

We will discuss these sections in more detail during the instructor-led lab.

Let's now deploy this configuration to Envoy and deploy our Envoy Proxy:

```bash
kubectl create cm envoy --from-file=envoy.yaml=./labs/01/envoy-conf.yaml -o yaml --dry-run=client | kubectl apply -f -

kubectl apply -f labs/01/envoy-proxy.yaml
```

Now let's try to call the Envoy Proxy and see that it correctly routes to the `httpbin` service:

<!--bash
until kubectl exec deploy/sleep -- curl http://envoy/headers -sSL -m 1 >/dev/null 2>&1
do
  sleep 1
done
-->

```bash
kubectl exec deploy/sleep -- curl http://envoy/headers
```

Now our response from `httpbin` should look similar to this:


```
{
  "headers": {
    "Accept": "*/*", 
    "Content-Length": "0", 
    "Host": "envoy", 
    "User-Agent": "curl/7.69.1", 
    "X-Envoy-Expected-Rq-Timeout-Ms": "15000"
  }
}
```

{% hint style="info" %}
We now see a response with some enriched response headers, `X-Envoy-Expected-Rq-Timeout-Ms`
{% endhint %}


Now that we have Envoy on the request path of a service-to-service interaction, let's try changing the behavior of the call. While in this default configuration, we saw an expected request timeout of `15s`, let's try to change the call timeout.


## Change the call timeout

To change the call timeout, let's take a look at the routing configuration and find the parameter that specified the timeout:

```
          routeConfig:
            name: simple_httpbin_route
            virtualHosts:
            - domains:
              - '*'
              name: httpbin_host
              routes:
              - match:
                  prefix: /
                route:
                  cluster: httpbin_service
                  timeout: 1s
```

Here, we can see we set the timeout to 1s. Let's try to call the `httpbin` service again through the Envoy proxy. First, let's update the configuration:

```bash
kubectl create cm envoy --from-file=envoy.yaml=./labs/01/envoy-conf-timeout.yaml -o yaml --dry-run=client | kubectl apply -f -
```
We will also need to _restart_ Envoy to pick up the new configuration:

```bash
kubectl rollout restart deploy/envoy
```
<!--bash
until kubectl exec deploy/sleep -- curl http://envoy/headers -sSL -m 1 >/dev/null 2>&1
do
  sleep 1
done
-->
```bash
kubectl exec deploy/sleep -- curl http://envoy/headers
```

We should see the headers now look like this:

```
{
  "headers": {
    "Accept": "*/*", 
    "Content-Length": "0", 
    "Host": "envoy", 
    "User-Agent": "curl/7.69.1", 
    "X-Envoy-Expected-Rq-Timeout-Ms": "1000"
  }
}
```

If we call our service, which takes longer than 1s, we should see a HTTP 504 / gateway timeout:

```bash
kubectl exec deploy/sleep -- curl -v http://envoy/delay/5
```

```
*   Trying 10.44.8.102:80...
* Connected to envoy (10.44.8.102) port 80 (#0)
> GET /delay/5 HTTP/1.1
> Host: envoy
> User-Agent: curl/7.69.1
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 504 Gateway Timeout
< content-length: 24
< content-type: text/plain
< date: Thu, 11 Feb 2021 22:28:33 GMT
< server: envoy
< 
* Connection #0 to host envoy left intact
upstream request timeout
    
```    

Although this is pretty simple so far, we can see how Envoy can become very valuable for a service-to-service request path. Enriching the networking with timeouts, retries, circuit breaking, etc allow the service to focus on business logic and differentiating features vs boring cross-cutting networking features.                            

## Admin stats

Now that we have a basic understanding of how to configure Envoy, let's take a look at another very important feature of Envoy proxy: stats and telemetry signals. One of the main reasons Envoy was even built was to give more visibility into what's happening on the network at the L7/request level. Let's take a look at how to get some of those stats:

Envoy exposes a `/stats` endpoint we can inspect:

```bash
kubectl exec deploy/sleep -- curl http://envoy:15000/stats
```

Wow, that's a lot of good info! Let's trim it down. Maybe we just want to see retry when the proxy calls the `httpbin` service

```bash
kubectl exec deploy/sleep -- curl http://envoy:15000/stats | grep retry
```

```
cluster.httpbin_service.circuit_breakers.default.rq_retry_open: 0
cluster.httpbin_service.circuit_breakers.high.rq_retry_open: 0
cluster.httpbin_service.retry_or_shadow_abandoned: 0
cluster.httpbin_service.upstream_rq_retry: 0
cluster.httpbin_service.upstream_rq_retry_backoff_exponential: 0
cluster.httpbin_service.upstream_rq_retry_backoff_ratelimited: 0
cluster.httpbin_service.upstream_rq_retry_limit_exceeded: 0
cluster.httpbin_service.upstream_rq_retry_overflow: 0
cluster.httpbin_service.upstream_rq_retry_success: 0
vhost.httpbin_host.vcluster.other.upstream_rq_retry: 0
vhost.httpbin_host.vcluster.other.upstream_rq_retry_limit_exceeded: 0
vhost.httpbin_host.vcluster.other.upstream_rq_retry_overflow: 0
vhost.httpbin_host.vcluster.other.upstream_rq_retry_success: 0
```

Nice! Things have been running smoothly so far. No request retries. Let's change that.

## Retrying failed requests

Calling services over the network can get scary. Services don't always respond, or may fail for some network reasons. We can have Envoy automatically retry when a request fails. Now, this isn't appropriate for every request, but [Envoy can be tuned](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_routing#arch-overview-http-routing-retry) to be smarter about when to retry.

Let's see how to configure to retry on HTTP `5xx` requests:

```
          routeConfig:
            name: simple_httpbin_route
            virtualHosts:
            - domains:
              - '*'
              name: httpbin_host
              routes:
              - match:
                  prefix: /
                route:
                  cluster: httpbin_service
                  timeout: 1s
                  retryPolicy:
                    retryOn: 5xx
                    numRetries: 3                      
```

{% hint style="info" %}
Note the configuration for retries on `5xx`
{% endhint %}


Let's apply this new configuration:

```bash
kubectl create cm envoy --from-file=envoy.yaml=./labs/01/envoy-conf-retry.yaml -o yaml --dry-run=client | kubectl apply -f -

kubectl rollout restart deploy/envoy
```

Now let's try to call an `httpbin` service endpoint that deliberately returns an error code:
<!--bash
until kubectl exec deploy/sleep -- curl http://envoy/status/200 -sSL -m 1 >/dev/null 2>&1
do
  sleep 1
done
-->
```bash
kubectl exec deploy/sleep -- curl -v http://envoy/status/500
```

We see the call fails:

```
*   Trying 10.44.8.102:80...
* Connected to envoy (10.44.8.102) port 80 (#0)
> GET /status/500 HTTP/1.1
> Host: envoy
> User-Agent: curl/7.69.1
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 500 Internal Server Error
< server: envoy
< date: Thu, 11 Feb 2021 23:34:32 GMT
< content-type: text/html; charset=utf-8
< access-control-allow-origin: *
< access-control-allow-credentials: true
< content-length: 0
< x-envoy-upstream-service-time: 130
< 
* Connection #0 to host envoy left intact
```

So let's see what Envoy observed in terms of retries:

```bash
kubectl exec deploy/sleep -- curl http://envoy:15000/stats | grep retry
```

```
cluster.httpbin_service.circuit_breakers.default.rq_retry_open: 0
cluster.httpbin_service.circuit_breakers.high.rq_retry_open: 0
cluster.httpbin_service.retry.upstream_rq_500: 3
cluster.httpbin_service.retry.upstream_rq_5xx: 3
cluster.httpbin_service.retry.upstream_rq_completed: 3
cluster.httpbin_service.retry_or_shadow_abandoned: 0
cluster.httpbin_service.upstream_rq_retry: 3
cluster.httpbin_service.upstream_rq_retry_backoff_exponential: 3
cluster.httpbin_service.upstream_rq_retry_backoff_ratelimited: 0
cluster.httpbin_service.upstream_rq_retry_limit_exceeded: 1
cluster.httpbin_service.upstream_rq_retry_overflow: 0
cluster.httpbin_service.upstream_rq_retry_success: 0
vhost.httpbin_host.vcluster.other.upstream_rq_retry: 0
vhost.httpbin_host.vcluster.other.upstream_rq_retry_limit_exceeded: 0
vhost.httpbin_host.vcluster.other.upstream_rq_retry_overflow: 0
vhost.httpbin_host.vcluster.other.upstream_rq_retry_success: 0
```

We see that indeed the call to `httpbin` did get retried 3 times. 

## Recap

So far we've taken a basic approach to understanding what the Envoy proxy is and how to configure it. We've also seen how it can alter the behavior of a network call and give us very valuable information about how the network is behaving at the request/message level. 

## Bonus Content

In the bonus section, we dig into more networking settings like load balancing, circuit breaking, and more. 

[See the Lab 01 bonus section](01a-bonus.md).

## Next Lab

In the [next lab](02-install-istio.md), we will dig into Istio's control plane a bit more. We'll also see how we can leverage all of these Envoy capabilities (resilience features, routing, observability, security, etc) to implement a secure, observable microservices architecture.                    
