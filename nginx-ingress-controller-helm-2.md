# Helm 3安装Nginx Ingress Controller

- ### 安装

先添加Chart仓库，注意是nginx-stable仓库，由nginx维护。

    helm repo add nginx-stable https://helm.nginx.com/stable

查找nginx-ingress，我们选择nginx-stable/nginx-ingress Chart。

    helm search repo nginx-ingress
    NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
    bitnami/nginx-ingress-controller        5.3.11          0.30.0          Chart for the nginx Ingress controller
    nginx-stable/nginx-ingress              0.4.3           1.6.3           NGINX Ingress Controller
    stable/nginx-ingress                    1.34.2          0.30.0          An nginx Ingress controller that uses ConfigMap...
    stable/nginx-lego                       0.3.1                           Chart for nginx-ingress-controller and kube-lego

展示values.yaml文件，分析helm安装Nginx Ingress的命令行覆盖参数。

    helm show values nginx-stable/nginx-ingress

由于Nginx Ingress的service缺省采用"type: LoadBalancer"，为了外部访问，修改为"type: NodePort"，顺便设置固定的nodePort。

    helm install gateway nginx-stable/nginx-ingress \
      --set controller.service.type=NodePort \
      --set controller.service.httpPort.nodePort=30080 \
      --set controller.service.httpsPort.nodePort=30443
    NAME: gateway
    LAST DEPLOYED: Fri Mar 27 17:53:01 2020
    NAMESPACE: default
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    The NGINX Ingress Controller has been installed.

查看Kubernetes资源。

    kubectl get all
    NAME                                         READY   STATUS    RESTARTS   AGE
    pod/gateway-nginx-ingress-55886df446-wx9zh   1/1     Running   0          26s
    
    NAME                            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
    service/gateway-nginx-ingress   NodePort    10.1.81.251   <none>        80:30080/TCP,443:30443/TCP   27s
    service/kubernetes              ClusterIP   10.1.0.1      <none>        443/TCP                      103d
    
    NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/gateway-nginx-ingress   1/1     1            1           27s
    
    NAME                                               DESIRED   CURRENT   READY   AGE
    replicaset.apps/gateway-nginx-ingress-55886df446   1         1         1       27s

尝试访问代理，本节点IP地址为192.168.1.55，代理端口为30080。由于没有配置路由规则，无法访问。

    curl http://192.168.1.55:30080
    <html>
    <head><title>404 Not Found</title></head>
    <body>
    <center><h1>404 Not Found</h1></center>
    <hr><center>nginx/1.17.9</center>
    </body>
    </html>

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

    vi echo-ingress.yaml
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

