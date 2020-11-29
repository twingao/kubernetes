# Helm 3安装Nginx Ingress Controller和使用

- ### 安装

查找nginx-ingress，我们选择`stable/nginx-ingress`。

    helm search repo nginx-ingress
    NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
    bitnami/nginx-ingress-controller        5.3.20          0.30.0          Chart for the nginx Ingress controller
    stable/nginx-ingress                    1.36.3          0.30.0          An nginx Ingress controller that uses ConfigMap...
    stable/nginx-lego                       0.3.1                           Chart for nginx-ingress-controller and kube-lego

展示values.yaml文件，分析helm安装Nginx Ingress的命令行覆盖参数。

    helm show values stable/nginx-ingress --version 1.36.3

由于Nginx Ingress的service缺省采用`type: LoadBalancer`，为了外部访问，修改为`type: NodePort`，顺便设置固定的nodePort。

    helm install gw stable/nginx-ingress \
      --version 1.36.3 \
      --namespace gateway \
      --set controller.service.type=NodePort \
      --set controller.service.nodePorts.http=30080 \
      --set controller.service.nodePorts.https=30443
    NAME: gw
    LAST DEPLOYED: Sun May 10 15:48:06 2020
    NAMESPACE: gateway
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    The nginx-ingress controller has been installed.
    Get the application URL by running these commands:
      export HTTP_NODE_PORT=30080
      export HTTPS_NODE_PORT=30443
      export NODE_IP=$(kubectl --namespace gateway get nodes -o jsonpath="{.items[0].status.addresses[1].address}")
    
      echo "Visit http://$NODE_IP:$HTTP_NODE_PORT to access your application via HTTP."
      echo "Visit https://$NODE_IP:$HTTPS_NODE_PORT to access your application via HTTPS."
    
    An example Ingress that makes use of the controller:
    
      apiVersion: extensions/v1beta1
      kind: Ingress
      metadata:
        annotations:
          kubernetes.io/ingress.class: nginx
        name: example
        namespace: foo
      spec:
        rules:
          - host: www.example.com
            http:
              paths:
                - backend:
                    serviceName: exampleService
                    servicePort: 80
                  path: /
        # This section is only required if TLS is to be enabled for the Ingress
        tls:
            - hosts:
                - www.example.com
              secretName: example-tls
    
    If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:
    
      apiVersion: v1
      kind: Secret
      metadata:
        name: example-tls
        namespace: foo
      data:
        tls.crt: <base64 encoded cert>
        tls.key: <base64 encoded key>
      type: kubernetes.io/tls

查看Kubernetes资源，注意到除了`gw-nginx-ingress-controller`服务，还有一个`gw-nginx-ingress-default-backend`服务，当nginx-ingress无法路由时就会分发到该服务处。

    kubectl get all -ngateway
    NAME                                                    READY   STATUS    RESTARTS   AGE
    pod/gw-nginx-ingress-controller-7cdd89986f-l6tq4        1/1     Running   0          7m36s
    pod/gw-nginx-ingress-default-backend-54c4d7cc5b-jfc69   1/1     Running   0          7m36s
    
    NAME                                       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
    service/gw-nginx-ingress-controller        NodePort    10.1.124.21   <none>        80:30080/TCP,443:30443/TCP   7m36s
    service/gw-nginx-ingress-default-backend   ClusterIP   10.1.250.34   <none>        80/TCP                       7m36s
    
    NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/gw-nginx-ingress-controller        1/1     1            1           7m36s
    deployment.apps/gw-nginx-ingress-default-backend   1/1     1            1           7m36s
    
    NAME                                                          DESIRED   CURRENT   READY   AGE
    replicaset.apps/gw-nginx-ingress-controller-7cdd89986f        1         1         1       7m36s
    replicaset.apps/gw-nginx-ingress-default-backend-54c4d7cc5b   1         1         1       7m36s

> 如果`k8s.gcr.io/defaultbackend-amd64:1.5`镜像无法下载，可以从此处下载。

> docker pull docker.io/twingao/defaultbackend-amd64:1.5

尝试访问代理，本节点IP地址为192.168.1.55，代理端口为30080。由于没有配置路由规则，无法访问，会分发到defaultbackend服务

    curl -i http://192.168.1.55:30080
    HTTP/1.1 404 Not Found
    Server: nginx/1.17.8
    Date: Sun, 10 May 2020 08:35:11 GMT
    Content-Type: text/plain; charset=utf-8
    Content-Length: 21
    Connection: keep-alive
    
    default backend - 404

- ### 使用

先创建两个集群服务。

    vi echo-service.yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: echo
      name: echo
    spec:
      ports:
      - name: http
        port: 8080
        protocol: TCP
        targetPort: 8080
      selector:
        app: echo
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: echo
      name: echo
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: echo
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: echo
        spec:
          containers:
          - image: e2eteam/echoserver:2.2
            name: echo
            ports:
            - containerPort: 8080
            env:
              - name: NODE_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: spec.nodeName
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
              - name: POD_NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
              - name: POD_IP
                valueFrom:
                  fieldRef:
                    fieldPath: status.podIP
            resources: {}
    
    vi httpbin-service.yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: httpbin
      labels:
        app: httpbin
    spec:
      ports:
      - name: http
        port: 80
        targetPort: 80
      selector:
        app: httpbin
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: httpbin
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: httpbin
      template:
        metadata:
          labels:
            app: httpbin
        spec:
          containers:
          - image: docker.io/kennethreitz/httpbin
            name: httpbin
            ports:
            - containerPort: 80
    
    kubectl apply -f echo-service.yaml
    kubectl apply -f httpbin-service.yaml

然后定义Ingress路由规则。

Ingress的namespace值为服务所在的命名空间，由于echo和httpbin服务在default命名空间，所以此处为default。

Ingress通过注解确定关联的Ingress Controller，注解`kubernetes.io/ingress.class: nginx`表示使用Nginx Ingress Controller。

    vi echo-ingress.yaml
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: echo-ingress
      namespace: default
      annotations:
        kubernetes.io/ingress.class: nginx
    spec:
      rules:
      - host: echo.com
        http:
          paths:
          - path: /
            backend:
              serviceName: echo
              servicePort: 8080
    
    vi httpbin-ingress.yaml
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: httpbin-ingress
      namespace: default
      annotations:
        kubernetes.io/ingress.class: nginx
    spec:
      rules:
      - host: httpbin.com
        http:
          paths:
          - path: /
            backend:
              serviceName: httpbin
              servicePort: 80

    kubectl apply -f echo-ingress.yaml
    kubectl apply -f httpbin-ingress.yaml

通过代理访问echo服务。

    curl -i -H "Host: echo.com" http://192.168.1.55:30080/
    HTTP/1.1 200 OK
    Server: nginx/1.17.9
    Date: Fri, 27 Mar 2020 11:13:40 GMT
    Content-Type: text/plain
    Transfer-Encoding: chunked
    Connection: keep-alive
    
    
    Hostname: echo-75cf96d976-bdh6l
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-75cf96d976-bdh6l
            pod namespace:  default
            pod IP: 10.244.1.3
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.2.4
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://echo.com:8080/
    
    Request Headers:
            accept=*/*
            connection=close
            host=echo.com
            user-agent=curl/7.29.0
            x-forwarded-for=10.244.0.0
            x-forwarded-host=echo.com
            x-forwarded-port=80
            x-forwarded-proto=http
            x-real-ip=10.244.0.0
    
    Request Body:
            -no body in request-

通过代理访问httpbin服务。

    curl -i -H "Host: httpbin.com" http://192.168.1.55:30080/status/200
    HTTP/1.1 200 OK
    Server: nginx/1.17.9
    Date: Fri, 27 Mar 2020 11:13:18 GMT
    Content-Type: text/html; charset=utf-8
    Content-Length: 0
    Connection: keep-alive
    Access-Control-Allow-Origin: *
    Access-Control-Allow-Credentials: true

删除两个Ingress。

    kubectl delete -f echo-ingress.yaml
    kubectl delete -f httpbin-ingress.yaml

将两个Ingress合并。

    vi echo-httpbin-ingress.yaml
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: echo-httpbin-ingress
      namespace: default
      annotations:
        kubernetes.io/ingress.class: nginx
    spec:
      rules:
      - host: echo.com
        http:
          paths:
          - path: /
            backend:
              serviceName: echo
              servicePort: 8080
      - host: httpbin.com
        http:
          paths:
          - path: /
            backend:
              serviceName: httpbin
              servicePort: 80
    
    kubectl apply -f echo-httpbin-ingress.yaml
    
    curl -i -H "Host: httpbin.com" http://192.168.1.55:30080/status/200
    HTTP/1.1 200 OK
    Server: nginx/1.17.8
    Date: Sun, 10 May 2020 08:38:47 GMT
    Content-Type: text/html; charset=utf-8
    Content-Length: 0
    Connection: keep-alive
    Access-Control-Allow-Origin: *
    Access-Control-Allow-Credentials: true
    
    curl -i -H "Host: echo.com" http://192.168.1.55:30080/
    HTTP/1.1 200 OK
    Server: nginx/1.17.8
    Date: Sun, 10 May 2020 08:38:57 GMT
    Content-Type: text/plain
    Transfer-Encoding: chunked
    Connection: keep-alive
    Vary: Accept-Encoding
    
    
    Hostname: echo-75cf96d976-hwjzv
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-75cf96d976-hwjzv
            pod namespace:  default
            pod IP: 10.244.1.4
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.1.3
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://echo.com:8080/
    
    Request Headers:
            accept=*/*
            host=echo.com
            user-agent=curl/7.29.0
            x-forwarded-for=10.244.0.0
            x-forwarded-host=echo.com
            x-forwarded-port=80
            x-forwarded-proto=http
            x-real-ip=10.244.0.0
            x-request-id=22e9d24e639106d6a3ca9b16d1819d75
            x-scheme=http
    
    Request Body:
            -no body in request-

app-root，好像没有效果。重定向了，但没有端口？

    vi echo-ingress.yaml
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: echo-ingress
      namespace: default
      annotations:
        kubernetes.io/ingress.class: nginx
        nginx.ingress.kubernetes.io/app-root: /echo
    spec:
      rules:
      - host: echo.com
        http:
          paths:
          - path: /
            backend:
              serviceName: echo
              servicePort: 8080
    
    curl -i -v -H "Host: echo.com" http://192.168.1.55:30080/
    * About to connect() to 192.168.1.55 port 30080 (#0)
    *   Trying 192.168.1.55...
    * Connected to 192.168.1.55 (192.168.1.55) port 30080 (#0)
    > GET / HTTP/1.1
    > User-Agent: curl/7.29.0
    > Accept: */*
    > Host: echo.com
    >
    < HTTP/1.1 302 Moved Temporarily
    HTTP/1.1 302 Moved Temporarily
    < Server: nginx/1.17.8
    Server: nginx/1.17.8
    < Date: Sun, 10 May 2020 12:49:51 GMT
    Date: Sun, 10 May 2020 12:49:51 GMT
    < Content-Type: text/html
    Content-Type: text/html
    < Content-Length: 145
    Content-Length: 145
    < Location: http://echo.com/echo
    Location: http://echo.com/echo
    < Connection: keep-alive
    Connection: keep-alive
    
    <
    <html>
    <head><title>302 Found</title></head>
    <body>
    <center><h1>302 Found</h1></center>
    <hr><center>nginx/1.17.8</center>
    </body>
    </html>
    * Connection #0 to host 192.168.1.55 left intact
    
    
    curl -i -v -H "Host: echo.com" http://192.168.1.55:30080/echo
    * About to connect() to 192.168.1.55 port 30080 (#0)
    *   Trying 192.168.1.55...
    * Connected to 192.168.1.55 (192.168.1.55) port 30080 (#0)
    > GET /echo HTTP/1.1
    > User-Agent: curl/7.29.0
    > Accept: */*
    > Host: echo.com
    >
    < HTTP/1.1 200 OK
    HTTP/1.1 200 OK
    < Server: nginx/1.17.8
    Server: nginx/1.17.8
    < Date: Sun, 10 May 2020 12:50:20 GMT
    Date: Sun, 10 May 2020 12:50:20 GMT
    < Content-Type: text/plain
    Content-Type: text/plain
    < Transfer-Encoding: chunked
    Transfer-Encoding: chunked
    < Connection: keep-alive
    Connection: keep-alive
    < Vary: Accept-Encoding
    Vary: Accept-Encoding
    
    <
    
    
    Hostname: echo-75cf96d976-hwjzv
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-75cf96d976-hwjzv
            pod namespace:  default
            pod IP: 10.244.1.4
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.1.3
            method=GET
            real path=/echo
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://echo.com:8080/echo
    
    Request Headers:
            accept=*/*
            host=echo.com
            user-agent=curl/7.29.0
            x-forwarded-for=10.244.0.0
            x-forwarded-host=echo.com
            x-forwarded-port=80
            x-forwarded-proto=http
            x-real-ip=10.244.0.0
            x-request-id=133fff1d8eef4e82d22b4562d7502ee5
            x-scheme=http
    
    Request Body:
            -no body in request-
    
    * Connection #0 to host 192.168.1.55 left intact
    
    
    curl -i -v -H "Host: echo.com" http://192.168.1.55:30080/e4dda
    * About to connect() to 192.168.1.55 port 30080 (#0)
    *   Trying 192.168.1.55...
    * Connected to 192.168.1.55 (192.168.1.55) port 30080 (#0)
    > GET /e4dda HTTP/1.1
    > User-Agent: curl/7.29.0
    > Accept: */*
    > Host: echo.com
    >
    < HTTP/1.1 200 OK
    HTTP/1.1 200 OK
    < Server: nginx/1.17.8
    Server: nginx/1.17.8
    < Date: Sun, 10 May 2020 12:50:56 GMT
    Date: Sun, 10 May 2020 12:50:56 GMT
    < Content-Type: text/plain
    Content-Type: text/plain
    < Transfer-Encoding: chunked
    Transfer-Encoding: chunked
    < Connection: keep-alive
    Connection: keep-alive
    < Vary: Accept-Encoding
    Vary: Accept-Encoding
    
    <
    
    
    Hostname: echo-75cf96d976-hwjzv
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-75cf96d976-hwjzv
            pod namespace:  default
            pod IP: 10.244.1.4
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.1.3
            method=GET
            real path=/e4dda
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://echo.com:8080/e4dda
    
    Request Headers:
            accept=*/*
            host=echo.com
            user-agent=curl/7.29.0
            x-forwarded-for=10.244.0.0
            x-forwarded-host=echo.com
            x-forwarded-port=80
            x-forwarded-proto=http
            x-real-ip=10.244.0.0
            x-request-id=145cfabc9b6b67504b55fcdc2393aac8
            x-scheme=http
    
    Request Body:
            -no body in request-
    
    * Connection #0 to host 192.168.1.55 left intact
    

路径中必须有前缀echo，如`http://echo.com/echo`可以，`http://echo.com/echo/ddd`可以，`http://echo.com/`不可以，`http://echo.com/asdf`不可以。


    vi echo-ingress.yaml
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: echo-ingress
      namespace: default
      annotations:
        kubernetes.io/ingress.class: "nginx"
        nginx.ingress.kubernetes.io/rewrite-target: /$2
    spec:
      rules:
      - host: echo.com
        http:
          paths:
          - path: /echo(/|$)(.*)
            backend:
              serviceName: echo
              servicePort: 8080

    curl -i -v -H "Host: echo.com" http://192.168.1.55:30080/echo
    * About to connect() to 192.168.1.55 port 30080 (#0)
    *   Trying 192.168.1.55...
    * Connected to 192.168.1.55 (192.168.1.55) port 30080 (#0)
    > GET /echo HTTP/1.1
    > User-Agent: curl/7.29.0
    > Accept: */*
    > Host: echo.com
    >
    < HTTP/1.1 200 OK
    HTTP/1.1 200 OK
    < Server: nginx/1.17.8
    Server: nginx/1.17.8
    < Date: Sun, 10 May 2020 13:02:11 GMT
    Date: Sun, 10 May 2020 13:02:11 GMT
    < Content-Type: text/plain
    Content-Type: text/plain
    < Transfer-Encoding: chunked
    Transfer-Encoding: chunked
    < Connection: keep-alive
    Connection: keep-alive
    < Vary: Accept-Encoding
    Vary: Accept-Encoding
    
    <
    
    
    Hostname: echo-75cf96d976-hwjzv
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-75cf96d976-hwjzv
            pod namespace:  default
            pod IP: 10.244.1.4
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.1.3
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://echo.com:8080/
    
    Request Headers:
            accept=*/*
            host=echo.com
            user-agent=curl/7.29.0
            x-forwarded-for=10.244.0.0
            x-forwarded-host=echo.com
            x-forwarded-port=80
            x-forwarded-proto=http
            x-real-ip=10.244.0.0
            x-request-id=881937ccc661cba4bbd3d5fdf086d512
            x-scheme=http
    
    Request Body:
            -no body in request-
    
    * Connection #0 to host 192.168.1.55 left intact
    
    curl -i -v -H "Host: echo.com" http://192.168.1.55:30080/echo/ddd
    * About to connect() to 192.168.1.55 port 30080 (#0)
    *   Trying 192.168.1.55...
    * Connected to 192.168.1.55 (192.168.1.55) port 30080 (#0)
    > GET /echo/ddd HTTP/1.1
    > User-Agent: curl/7.29.0
    > Accept: */*
    > Host: echo.com
    >
    < HTTP/1.1 200 OK
    HTTP/1.1 200 OK
    < Server: nginx/1.17.8
    Server: nginx/1.17.8
    < Date: Sun, 10 May 2020 13:11:37 GMT
    Date: Sun, 10 May 2020 13:11:37 GMT
    < Content-Type: text/plain
    Content-Type: text/plain
    < Transfer-Encoding: chunked
    Transfer-Encoding: chunked
    < Connection: keep-alive
    Connection: keep-alive
    < Vary: Accept-Encoding
    Vary: Accept-Encoding
    
    <
    
    
    Hostname: echo-75cf96d976-hwjzv
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-75cf96d976-hwjzv
            pod namespace:  default
            pod IP: 10.244.1.4
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.1.3
            method=GET
            real path=/ddd
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://echo.com:8080/ddd
    
    Request Headers:
            accept=*/*
            host=echo.com
            user-agent=curl/7.29.0
            x-forwarded-for=10.244.0.0
            x-forwarded-host=echo.com
            x-forwarded-port=80
            x-forwarded-proto=http
            x-real-ip=10.244.0.0
            x-request-id=3424f018174f2b3efab7e380aae05cd3
            x-scheme=http
    
    Request Body:
            -no body in request-
    
    * Connection #0 to host 192.168.1.55 left intact
    
    curl -i -v -H "Host: echo.com" http://192.168.1.55:30080/
    * About to connect() to 192.168.1.55 port 30080 (#0)
    *   Trying 192.168.1.55...
    * Connected to 192.168.1.55 (192.168.1.55) port 30080 (#0)
    > GET / HTTP/1.1
    > User-Agent: curl/7.29.0
    > Accept: */*
    > Host: echo.com
    >
    < HTTP/1.1 404 Not Found
    HTTP/1.1 404 Not Found
    < Server: nginx/1.17.8
    Server: nginx/1.17.8
    < Date: Sun, 10 May 2020 13:02:21 GMT
    Date: Sun, 10 May 2020 13:02:21 GMT
    < Content-Type: text/plain; charset=utf-8
    Content-Type: text/plain; charset=utf-8
    < Content-Length: 21
    Content-Length: 21
    < Connection: keep-alive
    Connection: keep-alive
    
    <
    * Connection #0 to host 192.168.1.55 left intact
    default backend - 404
    
    curl -i -v -H "Host: echo.com" http://192.168.1.55:30080/adsf
    * About to connect() to 192.168.1.55 port 30080 (#0)
    *   Trying 192.168.1.55...
    * Connected to 192.168.1.55 (192.168.1.55) port 30080 (#0)
    > GET /adsf HTTP/1.1
    > User-Agent: curl/7.29.0
    > Accept: */*
    > Host: echo.com
    >
    < HTTP/1.1 404 Not Found
    HTTP/1.1 404 Not Found
    < Server: nginx/1.17.8
    Server: nginx/1.17.8
    < Date: Sun, 10 May 2020 13:02:27 GMT
    Date: Sun, 10 May 2020 13:02:27 GMT
    < Content-Type: text/plain; charset=utf-8
    Content-Type: text/plain; charset=utf-8
    < Content-Length: 21
    Content-Length: 21
    < Connection: keep-alive
    Connection: keep-alive
    
    <
    * Connection #0 to host 192.168.1.55 left intact


将前缀去掉，`http://echo.com/echo` --> `http://echo.com/`；`http://echo.com/echo/ddd` --> `http://echo.com/ddd`; `http://echo.com/asdf/ddd` 不行。

    curl -i -v -H "Host: echo.com" http://192.168.1.55:30080/echo
    * About to connect() to 192.168.1.55 port 30080 (#0)
    *   Trying 192.168.1.55...
    * Connected to 192.168.1.55 (192.168.1.55) port 30080 (#0)
    > GET /echo HTTP/1.1
    > User-Agent: curl/7.29.0
    > Accept: */*
    > Host: echo.com
    >
    < HTTP/1.1 200 OK
    HTTP/1.1 200 OK
    < Server: nginx/1.17.8
    Server: nginx/1.17.8
    < Date: Sun, 10 May 2020 13:18:38 GMT
    Date: Sun, 10 May 2020 13:18:38 GMT
    < Content-Type: text/plain
    Content-Type: text/plain
    < Transfer-Encoding: chunked
    Transfer-Encoding: chunked
    < Connection: keep-alive
    Connection: keep-alive
    < Vary: Accept-Encoding
    Vary: Accept-Encoding
    
    <
    
    
    Hostname: echo-75cf96d976-hwjzv
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-75cf96d976-hwjzv
            pod namespace:  default
            pod IP: 10.244.1.4
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.1.3
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://echo.com:8080/
    
    Request Headers:
            accept=*/*
            host=echo.com
            user-agent=curl/7.29.0
            x-forwarded-for=10.244.0.0
            x-forwarded-host=echo.com
            x-forwarded-port=80
            x-forwarded-proto=http
            x-real-ip=10.244.0.0
            x-request-id=e84c814c690c537045f45f6e949a7f63
            x-scheme=http
    
    Request Body:
            -no body in request-
    
    * Connection #0 to host 192.168.1.55 left intact
    
    curl -i -v -H "Host: echo.com" http://192.168.1.55:30080/echo/ddd
    * About to connect() to 192.168.1.55 port 30080 (#0)
    *   Trying 192.168.1.55...
    * Connected to 192.168.1.55 (192.168.1.55) port 30080 (#0)
    > GET /echo/ddd HTTP/1.1
    > User-Agent: curl/7.29.0
    > Accept: */*
    > Host: echo.com
    >
    < HTTP/1.1 200 OK
    HTTP/1.1 200 OK
    < Server: nginx/1.17.8
    Server: nginx/1.17.8
    < Date: Sun, 10 May 2020 13:19:17 GMT
    Date: Sun, 10 May 2020 13:19:17 GMT
    < Content-Type: text/plain
    Content-Type: text/plain
    < Transfer-Encoding: chunked
    Transfer-Encoding: chunked
    < Connection: keep-alive
    Connection: keep-alive
    < Vary: Accept-Encoding
    Vary: Accept-Encoding
    
    <
    
    
    Hostname: echo-75cf96d976-hwjzv
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-75cf96d976-hwjzv
            pod namespace:  default
            pod IP: 10.244.1.4
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.1.3
            method=GET
            real path=/ddd
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://echo.com:8080/ddd
    
    Request Headers:
            accept=*/*
            host=echo.com
            user-agent=curl/7.29.0
            x-forwarded-for=10.244.0.0
            x-forwarded-host=echo.com
            x-forwarded-port=80
            x-forwarded-proto=http
            x-real-ip=10.244.0.0
            x-request-id=ba4902f9415ad8b734f4d06caf8a9d53
            x-scheme=http
    
    Request Body:
            -no body in request-
    
    * Connection #0 to host 192.168.1.55 left intact
    
    curl -i -v -H "Host: echo.com" http://192.168.1.55:30080/dddo/ddd
    * About to connect() to 192.168.1.55 port 30080 (#0)
    *   Trying 192.168.1.55...
    * Connected to 192.168.1.55 (192.168.1.55) port 30080 (#0)
    > GET /dddo/ddd HTTP/1.1
    > User-Agent: curl/7.29.0
    > Accept: */*
    > Host: echo.com
    >
    < HTTP/1.1 404 Not Found
    HTTP/1.1 404 Not Found
    < Server: nginx/1.17.8
    Server: nginx/1.17.8
    < Date: Sun, 10 May 2020 13:19:40 GMT
    Date: Sun, 10 May 2020 13:19:40 GMT
    < Content-Type: text/plain; charset=utf-8
    Content-Type: text/plain; charset=utf-8
    < Content-Length: 21
    Content-Length: 21
    < Connection: keep-alive
    Connection: keep-alive
    
    <
    * Connection #0 to host 192.168.1.55 left intact















重写url，如访问`http://prom.twingao.com/prom`，请求分发到服务是将路径删除，直接访问http://服务名称/。我们以Prometheus为例。

将Prometheus安装到monitoring命名空间。

    kubectl create namespace monitoring
    
    helm install mon stable/prometheus \
      --version 11.1.4 \
      --namespace monitoring \
      --set alertmanager.persistentVolume.enabled=false \
      --set server.persistentVolume.enabled=false
    
    kubectl get all -nmonitoring
    NAME                                              READY   STATUS    RESTARTS   AGE
    pod/mon-kube-state-metrics-5f994f45b4-rgl6m       1/1     Running   0          91s
    pod/mon-prometheus-alertmanager-99dcdff58-rhsx2   2/2     Running   0          91s
    pod/mon-prometheus-node-exporter-d5s6f            1/1     Running   0          91s
    pod/mon-prometheus-node-exporter-rthx8            1/1     Running   0          91s
    pod/mon-prometheus-pushgateway-6dbdf54bfc-s5d72   1/1     Running   0          91s
    pod/mon-prometheus-server-89b968bcc-vmwq5         1/2     Running   0          91s
    
    NAME                                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
    service/mon-kube-state-metrics         ClusterIP   10.1.195.144   <none>        8080/TCP   91s
    service/mon-prometheus-alertmanager    ClusterIP   10.1.28.46     <none>        80/TCP     91s
    service/mon-prometheus-node-exporter   ClusterIP   None           <none>        9100/TCP   91s
    service/mon-prometheus-pushgateway     ClusterIP   10.1.221.111   <none>        9091/TCP   91s
    service/mon-prometheus-server          ClusterIP   10.1.224.117   <none>        80/TCP     91s
    
    NAME                                          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
    daemonset.apps/mon-prometheus-node-exporter   2         2         2       2            2           <none>          91s
    
    NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/mon-kube-state-metrics        1/1     1            1           91s
    deployment.apps/mon-prometheus-alertmanager   1/1     1            1           91s
    deployment.apps/mon-prometheus-pushgateway    1/1     1            1           91s
    deployment.apps/mon-prometheus-server         0/1     1            0           91s
    
    NAME                                                    DESIRED   CURRENT   READY   AGE
    replicaset.apps/mon-kube-state-metrics-5f994f45b4       1         1         1       91s
    replicaset.apps/mon-prometheus-alertmanager-99dcdff58   1         1         1       91s
    replicaset.apps/mon-prometheus-pushgateway-6dbdf54bfc   1         1         1       91s
    replicaset.apps/mon-prometheus-server-89b968bcc         1         1         0       91s

访问Prometheus服务，可以看到被重定向到`http://10.1.224.117/graph`，可以在加-L参数试试。

    curl -i http://10.1.224.117
    HTTP/1.1 302 Found
    Content-Type: text/html; charset=utf-8
    Location: /graph
    Date: Sun, 10 May 2020 09:30:04 GMT
    Content-Length: 29
    
    <a href="/graph">Found</a>.

    curl -i -L http://10.1.224.117

配置Ingress



