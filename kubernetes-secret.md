# ConfigMap详解


Secret对象与ConfigMap对象类似，但它主要用于存储以下敏感信息，例如密码，OAuth token和SSH key等等。将这些信息存储在secret中，和直接存储在Pod的定义中，或Docker镜像定义中相比，更加安全和灵活。

1. Secret类型
Secret有三种类型：

- Opaque：使用base64编码存储信息，可以通过base64 --decode解码获得原始数据，因此安全性弱。
- kubernetes.io/dockerconfigjson：用于存储docker registry的认证信息。
- kubernetes.io/service-account-token：用于被 serviceaccount 引用。serviceaccout 创建时 Kubernetes 会默认创建对应的 secret。Pod 如果使用了 serviceaccount，对应的 secret 会自动挂载到 Pod 的 /run/secrets/kubernetes.io/serviceaccount 目录中。



    echo -n "admin" | base64
    YWRtaW4=
    echo -n "123456" | base64
    MTIzNDU2
    
    vi mysecret.yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: mysecret
    type: Opaque
    data:
      username: YWRtaW4=
      password: MTIzNDU2
    
    kubectl apply -f mysecret.yaml
    
    kubectl get secret mysecret -oyaml
    apiVersion: v1
    data:
      password: MTIzNDU2
      username: YWRtaW4=
    kind: Secret
    metadata:
      annotations:
        kubectl.kubernetes.io/last-applied-configuration: |
          {"apiVersion":"v1","data":{"password":"MTIzNDU2","username":"YWRtaW4="},"kind":"Secret","metadata":{"annotations":{},"name":"mysecret","namespace":"default"},"type":"Opaque"}
      creationTimestamp: "2020-04-07T23:27:08Z"
      name: mysecret
      namespace: default
      resourceVersion: "344759"
      selfLink: /api/v1/namespaces/default/secrets/mysecret
      uid: 984e7235-10ef-44d9-a33d-8c275c8bc84c
    type: Opaque
    
    
    
    etcdctl get /registry --prefix --keys-only=true | grep mysecret
    
    etcdctl get /registry/secrets/default/mysecret --prefix -w=fields
    "ClusterID" : 13595082567554063233
    "MemberID" : 13154517069409364498
    "Revision" : 346857
    "RaftTerm" : 8
    "Key" : "/registry/secrets/default/mysecret"
    "CreateRevision" : 344759
    "ModRevision" : 344759
    "Version" : 1
    "Value" : "k8s\x00\n\f\n\x02v1\x12\x06Secret\x12\xe6\x02\n\xb4\x02\n\bmysecret\x12\x00\x1a\adefault\"\x00*$984e7235-10ef-44d9-a33d-8c275c8bc84c2\x008\x00B\b\b̘\xb4\xf4\x05\x10\x00b\xe4\x01\n0kubectl.kubernetes.io/last-applied-configuration\x12\xaf\x01{\"apiVersion\":\"v1\",\"data\":{\"password\":\"MTIzNDU2\",\"username\":\"YWRtaW4=\"},\"kind\":\"Secret\",\"metadata\":{\"annotations\":{},\"name\":\"mysecret\",\"namespace\":\"default\"},\"type\":\"Opaque\"}\nz\x00\x12\x12\n\bpassword\x12\x06123456\x12\x11\n\busername\x12\x05admin\x1a\x06Opaque\x1a\x00\"\x00"
    "Lease" : 0
    "More" : false
    "Count" : 1





或者

echo -n "admin" > ./username.txt
echo -n "1f2d1e2e67df" > ./password.txt

kubectl create secret generic mysecret --from-file=./username.txt --from-file=./password.txt
secret "db-user-pass" created






ConfigMap是用来存储配置文件的kubernetes资源对象，所有的配置内容都存储在etcd中。

### 创建ConfigMap

创建ConfigMap的方式有4种：

- 通过直接在命令行中指定configmap参数创建，即--from-literal
- 通过指定文件创建，即将一个配置文件创建为一个ConfigMap--from-file=<文件>
- 通过指定目录创建，即将一个目录下的所有配置文件创建为一个ConfigMap，--from-file=<目录>
- 事先写好标准的configmap的yaml文件，然后kubectl create -f 创建。

#### from-literal

    kubectl create configmap test-config1 --from-literal=db.host=10.5.10.116 --from-literal=db.port=3306

    kubectl get configmap test-config1 -oyaml
    apiVersion: v1
    data:
      db.host: 10.5.10.116
      db.port: "3306"
    kind: ConfigMap
    metadata:
      creationTimestamp: "2020-04-05T11:07:59Z"
      name: test-config1
      namespace: default
      resourceVersion: "34029"
      selfLink: /api/v1/namespaces/default/configmaps/test-config1
      uid: 8aa68ffd-37b6-4ad8-ad6f-f5334c8d1fbe

data块为configmap的内容，其中有两个键值对，`db.host: 10.5.10.116`和`db.port: "3306"`。

可以查询test-config1在etcd中存放的Key。

    etcdctl get /registry --prefix --keys-only=true | grep test-config
    /registry/configmaps/default/test-config1

可以根据Key查询值。

    etcdctl get /registry/configmaps/default/test-config1 --prefix -w=fields
    "ClusterID" : 13595082567554063233
    "MemberID" : 13154517069409364498
    "Revision" : 36271
    "RaftTerm" : 8
    "Key" : "/registry/configmaps/default/test-config1"
    "CreateRevision" : 34029
    "ModRevision" : 34029
    "Version" : 1
    "Value" : "k8s\x00\n\x0f\n\x02v1\x12\tConfigMap\x12|\nQ\n\ftest-config1\x12\x00\x1a\adefault\"\x00*$8aa68ffd-37b6-4ad8-ad6f-f5334c8d1fbe2\x008\x00B\b\b\x8f\xf8\xa6\xf4\x05\x10\x00z\x00\x12\x16\n\adb.host\x12\v10.5.10.116\x12\x0f\n\adb.port\x12\x043306\x1a\x00\"\x00"
    "Lease" : 0
    "More" : false
    "Count" : 1

#### 指定文件创建

配置文件app.properties的内容：

    vi app.properties
    property.1 = value1
    property.2 = value2
    property.3 = value3
    
    # mysqld配置
    [mysqld]
    port      = 3309
    socket    = /root/mysql/tmp/mysql.sock
    pid-file  = /root/mysql/tmp/mysql.pid
    
    kubectl create configmap test-config2 --from-file=./app.properties
    
    kubectl get configmap test-config2 -oyaml
    apiVersion: v1
    data:
      app.properties: |
        property.1 = value1
        property.2 = value2
        property.3 = value3
    
        # mysqld配置
        [mysqld]
        port      = 3309
        socket    = /root/mysql/tmp/mysql.sock
        pid-file  = /root/mysql/tmp/mysql.pid
    kind: ConfigMap
    metadata:
      creationTimestamp: "2020-04-07T11:55:05Z"
      name: test-config2
      namespace: default
      resourceVersion: "285360"
      selfLink: /api/v1/namespaces/default/configmaps/test-config2
      uid: cfab06eb-cd3c-414d-8f46-cefcf8cba889

可以看到只有一个键值对，其中文件名app.properties为key，其值为文件内容。

也可以不适用文件名作为key，可以指定一个key。

    kubectl create configmap test-config3 --from-file=my-key-name=./app.properties
    
    kubectl get configmap test-config3 -oyaml
    apiVersion: v1
    data:
      my-key-name: |
        property.1 = value1
        property.2 = value2
        property.3 = value3
    
        # mysqld配置
        [mysqld]
        port      = 3309
        socket    = /root/mysql/tmp/mysql.sock
        pid-file  = /root/mysql/tmp/mysql.pid
    kind: ConfigMap
    metadata:
      creationTimestamp: "2020-04-07T14:23:14Z"
      name: test-config3
      namespace: default
      resourceVersion: "298077"
      selfLink: /api/v1/namespaces/default/configmaps/test-config3
      uid: 50f9e246-3e8c-47fd-9ab7-9312e66827f4

#### 指定目录创建

    mkdir configs
    cd configs/
    
    vi config1
    aaaaa
    bbbbb
    cc=dd

    vi config2
    eeeee
    fffff
    gg=hh

    cd ..
    kubectl create configmap test-config4 --from-file=./configs

    kubectl get configmap test-config4 -oyaml
    apiVersion: v1
    data:
      config1: |
        aaaaa
        bbbbb
        cc=dd
      config2: |
        eeeee
        fffff
        gg=hh
    kind: ConfigMap
    metadata:
      creationTimestamp: "2020-04-07T15:02:32Z"
      name: test-config4
      namespace: default
      resourceVersion: "301451"
      selfLink: /api/v1/namespaces/default/configmaps/test-config4
      uid: 5b4b2d02-940d-44d0-87d1-2f89b1386944

可以看到指定目录创建时，回味每个文件创建key/value对，key是文件名，value是文件内容。

注意，不支持递归，该目录下的子目录会被忽略。

### 通过yaml文件创建

    vi config6.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: config6
    data:
      apploglevel: info
      appdatadir: /var/data
    
    kubectl apply -f config6.yaml
    
    kubectl get configmap config6 -oyaml
    apiVersion: v1
    data:
      appdatadir: /var/data
      apploglevel: info
    kind: ConfigMap
    metadata:
      annotations:
        kubectl.kubernetes.io/last-applied-configuration: |
          {"apiVersion":"v1","data":{"appdatadir":"/var/data","apploglevel":"info"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"config6","namespace":"default"}}
      creationTimestamp: "2020-04-07T15:08:15Z"
      name: config6
      namespace: default
      resourceVersion: "301942"
      selfLink: /api/v1/namespaces/default/configmaps/config6
      uid: 92863846-9f5c-491d-b13d-0d653f47ec65
    
