# OpenTracing

启用NGINX服务的请求，以通过OpenTracing项目进行分布式跟踪.

使用第三方模块 [opentracing-contrib/nginx-opentracing](https://github.com/opentracing-contrib/nginx-opentracing) nginx ingress controller可以将NGINX配置为启用 [OpenTracing](http://opentracing.io) 检测。默认情况下，此功能处于禁用状态。

## Usage

要启用检测，我们必须在配置ConfigMap中启用OpenTracing
```
data:
  enable-opentracing: "true"
```

我们还必须设置上传跟踪时要使用的主机:

```
zipkin-collector-host: zipkin.default.svc.cluster.local
jaeger-collector-host: jaeger-agent.default.svc.cluster.local
datadog-collector-host: datadog-agent.default.svc.cluster.local
```
注意:在调用该选项 `jaeger-collector-host`时, 您需要将其指向 `jaeger-agent`, 而不是 `jaeger-collector` 组件.  

接下来，您将需要部署一个使用OpenTracing的分布式跟踪系统.
[Zipkin](https://github.com/openzipkin/zipkin) and
[Jaeger](https://github.com/jaegertracing/jaeger) and
[Datadog](https://github.com/DataDog/dd-opentracing-cpp)
have been tested.

其他可选配置选项:
```
# specifies the port to use when uploading traces, Default: 9411
zipkin-collector-port

# specifies the service name to use for any traces created, Default: nginx
zipkin-service-name

# specifies sample rate for any traces created, Default: 1.0
zipkin-sample-rate

# specifies the port to use when uploading traces, Default: 6831
jaeger-collector-port

# specifies the service name to use for any traces created, Default: nginx
jaeger-service-name

# specifies the sampler to be used when sampling traces.
# The available samplers are: const, probabilistic, ratelimiting, remote, Default: const
jaeger-sampler-type

# specifies the argument to be passed to the sampler constructor, Default: 1
jaeger-sampler-param

# Specifies the custom remote sampler host to be passed to the sampler constructor. Must be a valid URL.
# Default: http://127.0.0.1
jaeger-sampler-host

# Specifies the custom remote sampler port to be passed to the sampler constructor. Must be a number. Default: 5778
jaeger-sampler-port

# Specifies the header name used for passing trace context. Must be a string. Default: uber-trace-id
jaeger-trace-context-header-name

# Specifies the header name used for force sampling. Must be a string. Default: jaeger-debug-id
jaeger-debug-header

# Specifies the header name used to submit baggage if there is no root span. Must be a string. Default: jaeger-baggage
jaeger-baggage-header

# Specifies the header prefix used to propagate baggage. Must be a string. Default: uberctx-
jaeger-tracer-baggage-header-prefix

# specifies the port to use when uploading traces, Default 8126
datadog-collector-port

# specifies the service name to use for any traces created, Default: nginx
datadog-service-name

# specifies the operation name to use for any traces collected, Default: nginx.handle
datadog-operation-name-override
```

All these options (including host) allow environment variables, such as `$HOSTNAME` or `$HOST_IP`. In the case of Jaeger, if you have a Jaeger agent running on each machine in your cluster, you can use something like `$HOST_IP` (which can be 'mounted' with the `status.hostIP` fieldpath, as described [here](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/#capabilities-of-the-downward-api)) to make sure traces will be sent to the local agent.

## Examples

The following examples show how to deploy and test different distributed tracing systems. These example can be performed
using Minikube.

### Zipkin

In the [rnburn/zipkin-date-server](https://github.com/rnburn/zipkin-date-server)
GitHub repository is an example of a dockerized date service. To install the example and Zipkin collector run:

```
kubectl create -f https://raw.githubusercontent.com/rnburn/zipkin-date-server/master/kubernetes/zipkin.yaml
kubectl create -f https://raw.githubusercontent.com/rnburn/zipkin-date-server/master/kubernetes/deployment.yaml
```

Also we need to configure the nginx controller ConfigMap with the required values:

```
$ echo '
apiVersion: v1
kind: ConfigMap
data:
  enable-opentracing: "true"
  zipkin-collector-host: zipkin.default.svc.cluster.local
metadata:
  name: nginx-configuration
  namespace: kube-system
' | kubectl replace -f -
```

In the Zipkin interface we can see the details:
![zipkin screenshot](../../images/zipkin-demo.png "zipkin collector screenshot")

### Jaeger

1. Enable Ingress addon in Minikube:
    ```
    $ minikube addons enable ingress
    ```

2. Add Minikube IP to /etc/hosts:
    ```
    $ echo "$(minikube ip) example.com" | sudo tee -a /etc/hosts
    ```

3. Apply a basic Service and Ingress Resource:
    ```
    # Create Echoheaders Deployment
    $ kubectl run echoheaders --image=k8s.gcr.io/echoserver:1.4 --replicas=1 --port=8080

    # Expose as a Cluster-IP
    $ kubectl expose deployment echoheaders --port=80 --target-port=8080 --name=echoheaders-x

    # Apply the Ingress Resource
    $ echo '
      apiVersion: extensions/v1beta1
      kind: Ingress
      metadata:
        name: echo-ingress
      spec:
        rules:
        - host: example.com
          http:
            paths:
            - backend:
                serviceName: echoheaders-x
                servicePort: 80
              path: /echo
      ' | kubectl apply -f -
    ```

4. Enable OpenTracing and set the jaeger-collector-host:
    ```
    $ echo '
      apiVersion: v1
      kind: ConfigMap
      data:
        enable-opentracing: "true"
        jaeger-collector-host: jaeger-agent.default.svc.cluster.local
      metadata:
        name: nginx-configuration
        namespace: kube-system
      ' | kubectl replace -f -
    ```

5. Apply the Jaeger All-In-One Template:
    ```
    $ kubectl apply -f https://raw.githubusercontent.com/jaegertracing/jaeger-kubernetes/master/all-in-one/jaeger-all-in-one-template.yml
    ```

6. Make a few requests to the Service:
    ```
    $ curl example.com/echo -d "meow"

    CLIENT VALUES:
    client_address=172.17.0.5
    command=POST
    real path=/echo
    query=nil
    request_version=1.1
    request_uri=http://example.com:8080/echo

    SERVER VALUES:
    server_version=nginx: 1.10.0 - lua: 10001

    HEADERS RECEIVED:
    accept=*/*
    connection=close
    content-length=4
    content-type=application/x-www-form-urlencoded
    host=example.com
    user-agent=curl/7.54.0
    x-forwarded-for=192.168.99.1
    x-forwarded-host=example.com
    x-forwarded-port=80
    x-forwarded-proto=http
    x-original-uri=/echo
    x-real-ip=192.168.99.1
    x-scheme=http
    BODY:
    meow
    ```

7. View the Jaeger UI:
    ```
    $ minikube service jaeger-query --url

    http://192.168.99.100:30183
    ```

    In the Jaeger interface we can see the details:
    ![jaeger screenshot](../../images/jaeger-demo.png "jaeger collector screenshot")
