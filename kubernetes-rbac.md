# Kubernetes安全机制

## 认证


    cat /etc/kubernetes/manifests/kube-apiserver.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      annotations:
        kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 192.168.1.50:6443
      creationTimestamp: null
      labels:
        component: kube-apiserver
        tier: control-plane
      name: kube-apiserver
      namespace: kube-system
    spec:
      containers:
      - command:
        - kube-apiserver
        - --advertise-address=192.168.1.50
        - --allow-privileged=true
        - --authorization-mode=Node,RBAC
        - --client-ca-file=/etc/kubernetes/pki/ca.crt
        - --enable-admission-plugins=NodeRestriction
        - --enable-bootstrap-token-auth=true
        - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
        - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
        - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
        - --etcd-servers=https://127.0.0.1:2379
        - --insecure-port=0
        - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
        - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
        - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
        - --requestheader-allowed-names=front-proxy-client
        - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
        - --requestheader-extra-headers-prefix=X-Remote-Extra-
        - --requestheader-group-headers=X-Remote-Group
        - --requestheader-username-headers=X-Remote-User
        - --secure-port=6443
        - --service-account-key-file=/etc/kubernetes/pki/sa.pub
        - --service-cluster-ip-range=10.1.0.0/16
        - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
        - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
        - --service-node-port-range=1-39999
        - --basic-auth-file=/etc/kubernetes/pki/basic_auth_file.csv
        image: registry.aliyuncs.com/google_containers/kube-apiserver:v1.18.0
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 8
          httpGet:
            host: 192.168.1.50
            path: /healthz
            port: 6443
            scheme: HTTPS
          initialDelaySeconds: 15
          timeoutSeconds: 15
        name: kube-apiserver
        resources:
          requests:
            cpu: 250m
        volumeMounts:
        - mountPath: /etc/ssl/certs
          name: ca-certs
          readOnly: true
        - mountPath: /etc/pki
          name: etc-pki
          readOnly: true
        - mountPath: /etc/kubernetes/pki
          name: k8s-certs
          readOnly: true
      hostNetwork: true
      priorityClassName: system-cluster-critical
      volumes:
      - hostPath:
          path: /etc/ssl/certs
          type: DirectoryOrCreate
        name: ca-certs
      - hostPath:
          path: /etc/pki
          type: DirectoryOrCreate
        name: etc-pki
      - hostPath:
          path: /etc/kubernetes/pki
          type: DirectoryOrCreate
        name: k8s-certs
    status: {}

`/etc/kubernetes/manifests/`就是存放静态Pod的目录。

静态pod介绍
在Kubernetes中有一个DaemonSet类型的pod，这种类型的pod可以在某个节点上长期运行，这种类型的pod就是静态pod。
静态pod直接由某个节点上的kubelet程序进行管理，不需要api server介入，静态pod也不需要关联任何RC，完全是由kubelet程序来监控，当kubelet发现静态pod停止掉的时候，重新启动静态pod。


### 静态密码认证Static Password File


设置apiserver的启动参数：--basic_auth_file=SOMEFILE，如果更改了文件中的密码，只有重新启动apiserver使 其重新生效。其文件的基本格式包含三列：passwork，username，userid。当使用此作为认证方式时，在对apiserver的http 请求中，增加一个Header字段：Authorization，将它的值设置为： Basic BASE64ENCODEDUSER:PASSWORD。



这里要注意的是，容器化的kube-apiserver访问的目录是有范围的，可以在yaml的volume中看到能访问的目录，所以basic_auth_file文件应放在可访问的范围内。
用户名、密码和 UID

    vi /etc/kubernetes/pki/basic_auth_file.csv
    admin,admin,1
    twingao,twingao,2
    
    
    vi /etc/kubernetes/manifests/kube-apiserver.yaml
      - --basic-auth-file=/etc/kubernetes/pki/basic_auth_file.csv


Pod会被自动重启。

    kubectl get pod -nkube-system
    NAME                                 READY   STATUS    RESTARTS   AGE
    coredns-7ff77c879f-8flq5             1/1     Running   1          30h
    coredns-7ff77c879f-xth7n             1/1     Running   1          30h
    etcd-k8s-master                      1/1     Running   1          30h
    kube-apiserver-k8s-master            1/1     Running   0          39s
    kube-controller-manager-k8s-master   1/1     Running   3          30h
    kube-flannel-ds-amd64-8k5pg          1/1     Running   1          30h
    kube-flannel-ds-amd64-8nft4          1/1     Running   1          30h
    kube-flannel-ds-amd64-w27q7          1/1     Running   1          30h
    kube-proxy-b6c2q                     1/1     Running   1          30h
    kube-proxy-bvrhx                     1/1     Running   1          30h
    kube-proxy-fk7tn                     1/1     Running   1          30h
    kube-scheduler-k8s-master            1/1     Running   3          30h



    curl https://192.168.1.50:6443/version --basic -u twingao:twingao -k
    {
      "major": "1",
      "minor": "18",
      "gitVersion": "v1.18.0",
      "gitCommit": "9e991415386e4cf155a24b1da15becaa390438d8",
      "gitTreeState": "clean",
      "buildDate": "2020-03-25T14:50:46Z",
      "goVersion": "go1.13.8",
      "compiler": "gc",
      "platform": "linux/amd64"
    }
    
    curl https://192.168.1.50:6443/version --basic -u twingao:twingao1 -k
    {
      "kind": "Status",
      "apiVersion": "v1",
      "metadata": {
    
      },
      "status": "Failure",
      "message": "Unauthorized",
      "reason": "Unauthorized",
      "code": 401
    }
    
    
    curl https://192.168.1.50:6443/api/v1/namespaces --basic -u twingao:twingao -k
    {
      "kind": "Status",
      "apiVersion": "v1",
      "metadata": {
    
      },
      "status": "Failure",
      "message": "namespaces is forbidden: User \"twingao\" cannot list resource \"namespaces\" in API group \"\" at the cluster scope",
      "reason": "Forbidden",
      "details": {
        "kind": "namespaces"
      },
      "code": 403
    }



### 静态Token认证Static Token File


Static Token File（静态Token认证），启用此认证方式是通过设置apiserver参数--token-auth-file=SOMEFILE。Token长期有效，更改Token列表后需要重启apiserver才能生效。

SOMEFILE是CSV格式文件，至少包含3列：Token、用户名、用户的UID，其次是可选的分组。每一行表示一个用户。例如：

token,name,uid,"group1,group2,group3"

当使用HTTPS客户端连接时，apiserver从HTTPS Header中提取身份Token。Token必须是一个字符序列，在HTTPS Header格式如下：

Authorization: Bearer 31ada4fd-adec-460c-809a-9e56ceb75269

从请求中提取身份Token，与静态Token文件中的用户逐一进行比较，匹配成功则认证通过，否则认证失败。认证通过后将匹配的用户信息用于后续的授权等。




    vi /etc/kubernetes/pki/token_auth_file.csv
    abcdefg,admin,1
    hijklmn,twingao,2

    vi /etc/kubernetes/manifests/kube-apiserver.yaml
      - --token-auth-file=/etc/kubernetes/pki/token_auth_file.csv

    curl https://192.168.1.50:6443/version -H 'Authorization: Bearer abcdefg' -k
    {
      "major": "1",
      "minor": "18",
      "gitVersion": "v1.18.0",
      "gitCommit": "9e991415386e4cf155a24b1da15becaa390438d8",
      "gitTreeState": "clean",
      "buildDate": "2020-03-25T14:50:46Z",
      "goVersion": "go1.13.8",
      "compiler": "gc",
      "platform": "linux/amd64"
    }
    
    
    curl https://192.168.1.50:6443/api/v1/namespaces -H 'Authorization: Bearer abcdefg' -k
    {
      "kind": "NamespaceList",
      "apiVersion": "v1",
      "metadata": {
        "selfLink": "/api/v1/namespaces",
        "resourceVersion": "302784"
      },
      "items": [
        {
          "metadata": {
            "name": "default",
            "selfLink": "/api/v1/namespaces/default",
            "uid": "54e38c8f-b349-4032-bd13-580da80bb439",
            "resourceVersion": "152",
            "creationTimestamp": "2020-04-08T02:10:55Z",
            "managedFields": [
              {
                "manager": "kube-apiserver",
                "operation": "Update",
                "apiVersion": "v1",
                "time": "2020-04-08T02:10:55Z",
                "fieldsType": "FieldsV1",
                "fieldsV1": {"f:status":{"f:phase":{}}}
              }
            ]
          },
          "spec": {
            "finalizers": [
              "kubernetes"
            ]
          },
          "status": {
            "phase": "Active"
          }
        },
        {
          "metadata": {
            "name": "kube-node-lease",
            "selfLink": "/api/v1/namespaces/kube-node-lease",
            "uid": "360398ba-444c-4936-9ff2-2b9e6ccd83a4",
            "resourceVersion": "42",
            "creationTimestamp": "2020-04-08T02:10:53Z",
            "managedFields": [
              {
                "manager": "kube-apiserver",
                "operation": "Update",
                "apiVersion": "v1",
                "time": "2020-04-08T02:10:53Z",
                "fieldsType": "FieldsV1",
                "fieldsV1": {"f:status":{"f:phase":{}}}
              }
            ]
          },
          "spec": {
            "finalizers": [
              "kubernetes"
            ]
          },
          "status": {
            "phase": "Active"
          }
        },
        {
          "metadata": {
            "name": "kube-public",
            "selfLink": "/api/v1/namespaces/kube-public",
            "uid": "373f499a-d5a8-4722-aebd-533e482a4d19",
            "resourceVersion": "41",
            "creationTimestamp": "2020-04-08T02:10:53Z",
            "managedFields": [
              {
                "manager": "kube-apiserver",
                "operation": "Update",
                "apiVersion": "v1",
                "time": "2020-04-08T02:10:53Z",
                "fieldsType": "FieldsV1",
                "fieldsV1": {"f:status":{"f:phase":{}}}
              }
            ]
          },
          "spec": {
            "finalizers": [
              "kubernetes"
            ]
          },
          "status": {
            "phase": "Active"
          }
        },
        {
          "metadata": {
            "name": "kube-system",
            "selfLink": "/api/v1/namespaces/kube-system",
            "uid": "9c6570eb-0893-42ab-914c-6562c784d1a3",
            "resourceVersion": "11",
            "creationTimestamp": "2020-04-08T02:10:53Z",
            "managedFields": [
              {
                "manager": "kube-apiserver",
                "operation": "Update",
                "apiVersion": "v1",
                "time": "2020-04-08T02:10:53Z",
                "fieldsType": "FieldsV1",
                "fieldsV1": {"f:status":{"f:phase":{}}}
              }
            ]
          },
          "spec": {
            "finalizers": [
              "kubernetes"
            ]
          },
          "status": {
            "phase": "Active"
          }
        }
      ]
    }
    
    curl https://192.168.1.50:6443/api/v1/namespaces -H 'Authorization: Bearer abcdefg1' -k
    {
      "kind": "Status",
      "apiVersion": "v1",
      "metadata": {
    
      },
      "status": "Failure",
      "message": "Unauthorized",
      "reason": "Unauthorized",
      "code": 401
    }









## 授权

### AlwaysAllow

    vi /etc/kubernetes/manifests/kube-apiserver.yaml
        - --authorization-mode=AlwaysAllow
    
    
    kubectl get pod -nkube-system
    NAME                                 READY   STATUS    RESTARTS   AGE
    coredns-7ff77c879f-8flq5             1/1     Running   1          36h
    coredns-7ff77c879f-xth7n             1/1     Running   1          36h
    etcd-k8s-master                      1/1     Running   1          36h
    kube-apiserver-k8s-master            1/1     Running   1          32s
    kube-controller-manager-k8s-master   1/1     Running   5          36h
    kube-flannel-ds-amd64-8k5pg          1/1     Running   1          35h
    kube-flannel-ds-amd64-8nft4          1/1     Running   1          35h
    kube-flannel-ds-amd64-w27q7          1/1     Running   1          35h
    kube-proxy-b6c2q                     1/1     Running   1          35h
    kube-proxy-bvrhx                     1/1     Running   1          35h
    kube-proxy-fk7tn                     1/1     Running   1          36h
    kube-scheduler-k8s-master            1/1     Running   5          36h
    
    
    curl https://192.168.1.50:6443/version --basic -u twingao:twingao -k
    {
      "major": "1",
      "minor": "18",
      "gitVersion": "v1.18.0",
      "gitCommit": "9e991415386e4cf155a24b1da15becaa390438d8",
      "gitTreeState": "clean",
      "buildDate": "2020-03-25T14:50:46Z",
      "goVersion": "go1.13.8",
      "compiler": "gc",
      "platform": "linux/amd64"
    }
    
    curl https://192.168.1.50:6443/api/v1/namespaces --basic -u twingao:twingao -k
    {
      "kind": "NamespaceList",
      "apiVersion": "v1",
      "metadata": {
        "selfLink": "/api/v1/namespaces",
        "resourceVersion": "300037"
      },
      "items": [
        {
          "metadata": {
            "name": "default",
            "selfLink": "/api/v1/namespaces/default",
            "uid": "54e38c8f-b349-4032-bd13-580da80bb439",
            "resourceVersion": "152",
            "creationTimestamp": "2020-04-08T02:10:55Z",
            "managedFields": [
              {
                "manager": "kube-apiserver",
                "operation": "Update",
                "apiVersion": "v1",
                "time": "2020-04-08T02:10:55Z",
                "fieldsType": "FieldsV1",
                "fieldsV1": {"f:status":{"f:phase":{}}}
              }
            ]
          },
          "spec": {
            "finalizers": [
              "kubernetes"
            ]
          },
          "status": {
            "phase": "Active"
          }
        },
        {
          "metadata": {
            "name": "kube-node-lease",
            "selfLink": "/api/v1/namespaces/kube-node-lease",
            "uid": "360398ba-444c-4936-9ff2-2b9e6ccd83a4",
            "resourceVersion": "42",
            "creationTimestamp": "2020-04-08T02:10:53Z",
            "managedFields": [
              {
                "manager": "kube-apiserver",
                "operation": "Update",
                "apiVersion": "v1",
                "time": "2020-04-08T02:10:53Z",
                "fieldsType": "FieldsV1",
                "fieldsV1": {"f:status":{"f:phase":{}}}
              }
            ]
          },
          "spec": {
            "finalizers": [
              "kubernetes"
            ]
          },
          "status": {
            "phase": "Active"
          }
        },
        {
          "metadata": {
            "name": "kube-public",
            "selfLink": "/api/v1/namespaces/kube-public",
            "uid": "373f499a-d5a8-4722-aebd-533e482a4d19",
            "resourceVersion": "41",
            "creationTimestamp": "2020-04-08T02:10:53Z",
            "managedFields": [
              {
                "manager": "kube-apiserver",
                "operation": "Update",
                "apiVersion": "v1",
                "time": "2020-04-08T02:10:53Z",
                "fieldsType": "FieldsV1",
                "fieldsV1": {"f:status":{"f:phase":{}}}
              }
            ]
          },
          "spec": {
            "finalizers": [
              "kubernetes"
            ]
          },
          "status": {
            "phase": "Active"
          }
        },
        {
          "metadata": {
            "name": "kube-system",
            "selfLink": "/api/v1/namespaces/kube-system",
            "uid": "9c6570eb-0893-42ab-914c-6562c784d1a3",
            "resourceVersion": "11",
            "creationTimestamp": "2020-04-08T02:10:53Z",
            "managedFields": [
              {
                "manager": "kube-apiserver",
                "operation": "Update",
                "apiVersion": "v1",
                "time": "2020-04-08T02:10:53Z",
                "fieldsType": "FieldsV1",
                "fieldsV1": {"f:status":{"f:phase":{}}}
              }
            ]
          },
          "spec": {
            "finalizers": [
              "kubernetes"
            ]
          },
          "status": {
            "phase": "Active"
          }
        }
      ]
    }


### AlwaysDeny

    vi /etc/kubernetes/manifests/kube-apiserver.yaml
        - --authorization-mode=AlwaysDeny
    
    kubectl get pod -nkube-system
    The connection to the server 192.168.1.50:6443 was refused - did you specify the right host or port?
    
    
    curl https://192.168.1.50:6443/version --basic -u twingao:twingao -k
    {
      "kind": "Status",
      "apiVersion": "v1",
      "metadata": {
    
      },
      "status": "Failure",
      "message": "forbidden: User \"twingao\" cannot get path \"/version\": Everything is forbidden.",
      "reason": "Forbidden",
      "details": {
    
      },
      "code": 403
    }
    
    curl https://192.168.1.50:6443/api/v1/namespaces --basic -u twingao:twingao -k
    {
      "kind": "Status",
      "apiVersion": "v1",
      "metadata": {
    
      },
      "status": "Failure",
      "message": "namespaces is forbidden: User \"twingao\" cannot list resource \"namespaces\" in API group \"\" at the cluster scope: Everything is forbidden.",
      "reason": "Forbidden",
      "details": {
        "kind": "namespaces"
      },
      "code": 403
    }


## curl -> api server

### serviceaccount

    kubectl create serviceaccount twingao
    
    kubectl get serviceaccount twingao -oyaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      creationTimestamp: "2020-04-10T02:26:28Z"
      name: twingao
      namespace: default
      resourceVersion: "312521"
      selfLink: /api/v1/namespaces/default/serviceaccounts/twingao
      uid: 2b91995e-d6d2-4ebb-80c4-dd1679656321
    secrets:
    - name: twingao-token-vlp7x
    
    vi podsvc-read.yaml
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1beta1
    metadata:
      name: podsvc-read
    rules:
    - apiGroups: [""]
      resources: ["pods", "services", "pods/log", "namespaces"]
      verbs: ["get", "watch", "list"]
    
    kubectl apply -f podsvc-read.yaml

    vi twingao-podsvc-read.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: twingao-podsvc-read
    subjects:
    - kind: ServiceAccount
      name: twingao
      apiGroup: ""
    roleRef:
      kind: ClusterRole
      name: podsvc-read
      apiGroup: ""
    
    kubectl apply -f twingao-podsvc-read.yaml
    
    
    kubectl get secret twingao-token-vlp7x -oyaml
    apiVersion: v1
    data:
      ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01EUXdPREF5TVRBeU1sb1hEVE13TURRd05qQXlNVEF5TWxvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTDZxCnZmaGtGdTdLQm5ReVBCMzNYUlovbXdIQ0lLL3FpakdMMkQrTHdJZEpFTGJDQk9QdWZPa2hFWkNWZTFVMFRoVjUKamJDbTc0cFBPNThycldFdVNFSXZ2ejlNcERDRWtoTDVhb1NDbUY2OU01WjNpMU41SmJheTUxeW05TERmR3YwcwppWVBkUS9acG9pYk4xRjl3ZzJTTU5nYjNwbjFZbS82REFWWUtqaHZFQzhyT09iWDJ5WUxBSWsxNFdiKy8zaUh1CkNEYzcrTnEzUjJLOGtkc0Q4RzdyTGVWS2krUHZHUDhPeStXUjcrelBSbEdLOWJtc1Rac3J5dUNpd09ad0hGRzUKS2o3cXhmM1pNd1hvS0dTcjdZSU16aEVrUkpTZFlTQUZGM0xLVFBoV2hlb3hwWWxWZlNuWXdJYUZpaS9CeHFHYgpuMnMza1pZUVpTYzd6eVNCRmcwQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFHd2NzVkpFNjVoUkJzS1h6WTQxVjBwQ2xXaWMKQUNZaHNuc3NkaEFaN1pqY3ZxMTVPZGIzZVh4ak1wTkw5eStsZU9vcTBPZ1hPQ3oyMGs0K21sRldFRFFxei9MbQpja3RPTHFISjd4aG9mU2xOYUROV2F1NlpsTW5DMFBOdmh4UzUvUWNZSHYzR2djbVVwdjJSUGU1eXB3VWt3bEdyCm1iOXNRTFdWNW5PNzRBNWs4VkhhWjVvME5YbSt1dDgxcTdMd2hzaW40MjRyTWw1Snh4OWQxbjRvVXJOS3ZyQUkKOFV5MjlrbXE3T0Z5eGVERlVRZExiR0ZMZFVaQ3JiM01zM0ZRQVpyZDZYTUxGZm5PT1ppc2NiUDBTK2dZMHJ0aQpxclpwNVIxQkJzNWpjcEUvZ1NXaCtDQmFDaE1GVnJKdVpUZTBHTzdYQkxUWXorTmZ1aDZ2SVZYS2ZNaz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
      namespace: ZGVmYXVsdA==
      token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNkltcHRjRE5yYVdKRlVFb3lNbVZmWWtkQmVWVkJObmhvUm5rNWVUbGlUVk5uT0V4bmRFWnZTRE5FVjBFaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJblIzYVc1bllXOHRkRzlyWlc0dGRteHdOM2dpTENKcmRXSmxjbTVsZEdWekxtbHZMM05sY25acFkyVmhZMk52ZFc1MEwzTmxjblpwWTJVdFlXTmpiM1Z1ZEM1dVlXMWxJam9pZEhkcGJtZGhieUlzSW10MVltVnlibVYwWlhNdWFXOHZjMlZ5ZG1salpXRmpZMjkxYm5RdmMyVnlkbWxqWlMxaFkyTnZkVzUwTG5WcFpDSTZJakppT1RFNU9UVmxMV1EyWkRJdE5HVmlZaTA0TUdNMExXUmtNVFkzT1RZMU5qTXlNU0lzSW5OMVlpSTZJbk41YzNSbGJUcHpaWEoyYVdObFlXTmpiM1Z1ZERwa1pXWmhkV3gwT25SM2FXNW5ZVzhpZlEua25lanF4M0wzNzJHek90cFg3NXFxOW9JeDNWQ1RVVm44Z1QxRWZ1SkFmOFB3bWFQc0YzUVBDbWJPVHpsbzRGSXh5NW9zbFk0YmI5bkVzOGlic1ZfZnBPdVh1UXRVbWtjNlRDSHJxMEw4Q1pYRGxfYkVnT2VXa3FsU0JmdU5Ub2MzVy1MZ0tmNTlvOEFaT3VGR0RmelNDQTBtelV5RktOUEdfS0hCMzRzOHdRcmJFeU8xM24tMnFkTkhuR29xTHFpOFFPSHRnR3ZvRkRiM3BSc0x3UWlPbHIyM0gxQ0dLa1VGY3BOYzdBS2l5QTZHZmZLX0xzTGxjLWYtVUg4ZlYweGlFbXJETFdqcnkwQlZPeXZPSmZCX1dpejcyT3d6cUsyZ3EtLVFWbVFTTUd2Nk44QjFiXzlnWVBZMzFmbFEwSUlpejlka2NQRUxsNUxaX19nbFhoV1FB
    kind: Secret
    metadata:
      annotations:
        kubernetes.io/service-account.name: twingao
        kubernetes.io/service-account.uid: 2b91995e-d6d2-4ebb-80c4-dd1679656321
      creationTimestamp: "2020-04-10T02:26:28Z"
      managedFields:
      - apiVersion: v1
        fieldsType: FieldsV1
        fieldsV1:
          f:data:
            .: {}
            f:ca.crt: {}
            f:namespace: {}
            f:token: {}
          f:metadata:
            f:annotations:
              .: {}
              f:kubernetes.io/service-account.name: {}
              f:kubernetes.io/service-account.uid: {}
          f:type: {}
        manager: kube-controller-manager
        operation: Update
        time: "2020-04-10T02:26:28Z"
      name: twingao-token-vlp7x
      namespace: default
      resourceVersion: "312520"
      selfLink: /api/v1/namespaces/default/secrets/twingao-token-vlp7x
      uid: af1e50eb-d842-4939-82ab-0aaa516e4d35
    type: kubernetes.io/service-account-token
    
    
    echo 'LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01EUXdPREF5TVRBeU1sb1hEVE13TURRd05qQXlNVEF5TWxvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTDZxCnZmaGtGdTdLQm5ReVBCMzNYUlovbXdIQ0lLL3FpakdMMkQrTHdJZEpFTGJDQk9QdWZPa2hFWkNWZTFVMFRoVjUKamJDbTc0cFBPNThycldFdVNFSXZ2ejlNcERDRWtoTDVhb1NDbUY2OU01WjNpMU41SmJheTUxeW05TERmR3YwcwppWVBkUS9acG9pYk4xRjl3ZzJTTU5nYjNwbjFZbS82REFWWUtqaHZFQzhyT09iWDJ5WUxBSWsxNFdiKy8zaUh1CkNEYzcrTnEzUjJLOGtkc0Q4RzdyTGVWS2krUHZHUDhPeStXUjcrelBSbEdLOWJtc1Rac3J5dUNpd09ad0hGRzUKS2o3cXhmM1pNd1hvS0dTcjdZSU16aEVrUkpTZFlTQUZGM0xLVFBoV2hlb3hwWWxWZlNuWXdJYUZpaS9CeHFHYgpuMnMza1pZUVpTYzd6eVNCRmcwQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFHd2NzVkpFNjVoUkJzS1h6WTQxVjBwQ2xXaWMKQUNZaHNuc3NkaEFaN1pqY3ZxMTVPZGIzZVh4ak1wTkw5eStsZU9vcTBPZ1hPQ3oyMGs0K21sRldFRFFxei9MbQpja3RPTHFISjd4aG9mU2xOYUROV2F1NlpsTW5DMFBOdmh4UzUvUWNZSHYzR2djbVVwdjJSUGU1eXB3VWt3bEdyCm1iOXNRTFdWNW5PNzRBNWs4VkhhWjVvME5YbSt1dDgxcTdMd2hzaW40MjRyTWw1Snh4OWQxbjRvVXJOS3ZyQUkKOFV5MjlrbXE3T0Z5eGVERlVRZExiR0ZMZFVaQ3JiM01zM0ZRQVpyZDZYTUxGZm5PT1ppc2NiUDBTK2dZMHJ0aQpxclpwNVIxQkJzNWpjcEUvZ1NXaCtDQmFDaE1GVnJKdVpUZTBHTzdYQkxUWXorTmZ1aDZ2SVZYS2ZNaz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=' | base64 -d > ca.crt
    
    echo 'ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNkltcHRjRE5yYVdKRlVFb3lNbVZmWWtkQmVWVkJObmhvUm5rNWVUbGlUVk5uT0V4bmRFWnZTRE5FVjBFaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJblIzYVc1bllXOHRkRzlyWlc0dGRteHdOM2dpTENKcmRXSmxjbTVsZEdWekxtbHZMM05sY25acFkyVmhZMk52ZFc1MEwzTmxjblpwWTJVdFlXTmpiM1Z1ZEM1dVlXMWxJam9pZEhkcGJtZGhieUlzSW10MVltVnlibVYwWlhNdWFXOHZjMlZ5ZG1salpXRmpZMjkxYm5RdmMyVnlkbWxqWlMxaFkyTnZkVzUwTG5WcFpDSTZJakppT1RFNU9UVmxMV1EyWkRJdE5HVmlZaTA0TUdNMExXUmtNVFkzT1RZMU5qTXlNU0lzSW5OMVlpSTZJbk41YzNSbGJUcHpaWEoyYVdObFlXTmpiM1Z1ZERwa1pXWmhkV3gwT25SM2FXNW5ZVzhpZlEua25lanF4M0wzNzJHek90cFg3NXFxOW9JeDNWQ1RVVm44Z1QxRWZ1SkFmOFB3bWFQc0YzUVBDbWJPVHpsbzRGSXh5NW9zbFk0YmI5bkVzOGlic1ZfZnBPdVh1UXRVbWtjNlRDSHJxMEw4Q1pYRGxfYkVnT2VXa3FsU0JmdU5Ub2MzVy1MZ0tmNTlvOEFaT3VGR0RmelNDQTBtelV5RktOUEdfS0hCMzRzOHdRcmJFeU8xM24tMnFkTkhuR29xTHFpOFFPSHRnR3ZvRkRiM3BSc0x3UWlPbHIyM0gxQ0dLa1VGY3BOYzdBS2l5QTZHZmZLX0xzTGxjLWYtVUg4ZlYweGlFbXJETFdqcnkwQlZPeXZPSmZCX1dpejcyT3d6cUsyZ3EtLVFWbVFTTUd2Nk44QjFiXzlnWVBZMzFmbFEwSUlpejlka2NQRUxsNUxaX19nbFhoV1FB' | base64 -d
    eyJhbGciOiJSUzI1NiIsImtpZCI6ImptcDNraWJFUEoyMmVfYkdBeVVBNnhoRnk5eTliTVNnOExndEZvSDNEV0EifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InR3aW5nYW8tdG9rZW4tdmxwN3giLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoidHdpbmdhbyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjJiOTE5OTVlLWQ2ZDItNGViYi04MGM0LWRkMTY3OTY1NjMyMSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OnR3aW5nYW8ifQ.knejqx3L372GzOtpX75qq9oIx3VCTUVn8gT1EfuJAf8PwmaPsF3QPCmbOTzlo4FIxy5oslY4bb9nEs8ibsV_fpOuXuQtUmkc6TCHrq0L8CZXDl_bEgOeWkqlSBfuNToc3W-LgKf59o8AZOuFGDfzSCA0mzUyFKNPG_KHB34s8wQrbEyO13n-2qdNHnGoqLqi8QOHtgGvoFDb3pRsLwQiOlr23H1CGKkUFcpNc7AKiyA6GffK_LsLlc-f-UH8fV0xiEmrDLWjry0BVOyvOJfB_Wiz72OwzqK2gq--QVmQSMGv6N8B1b_9gYPY31flQ0IIiz9dkcPELl5LZ__glXhWQA
    
    echo 'ZGVmYXVsdA==' | base64 -d
    default
    
    
    cat /root/.kube/config
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01EUXdPREF5TVRBeU1sb1hEVE13TURRd05qQXlNVEF5TWxvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTDZxCnZmaGtGdTdLQm5ReVBCMzNYUlovbXdIQ0lLL3FpakdMMkQrTHdJZEpFTGJDQk9QdWZPa2hFWkNWZTFVMFRoVjUKamJDbTc0cFBPNThycldFdVNFSXZ2ejlNcERDRWtoTDVhb1NDbUY2OU01WjNpMU41SmJheTUxeW05TERmR3YwcwppWVBkUS9acG9pYk4xRjl3ZzJTTU5nYjNwbjFZbS82REFWWUtqaHZFQzhyT09iWDJ5WUxBSWsxNFdiKy8zaUh1CkNEYzcrTnEzUjJLOGtkc0Q4RzdyTGVWS2krUHZHUDhPeStXUjcrelBSbEdLOWJtc1Rac3J5dUNpd09ad0hGRzUKS2o3cXhmM1pNd1hvS0dTcjdZSU16aEVrUkpTZFlTQUZGM0xLVFBoV2hlb3hwWWxWZlNuWXdJYUZpaS9CeHFHYgpuMnMza1pZUVpTYzd6eVNCRmcwQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFHd2NzVkpFNjVoUkJzS1h6WTQxVjBwQ2xXaWMKQUNZaHNuc3NkaEFaN1pqY3ZxMTVPZGIzZVh4ak1wTkw5eStsZU9vcTBPZ1hPQ3oyMGs0K21sRldFRFFxei9MbQpja3RPTHFISjd4aG9mU2xOYUROV2F1NlpsTW5DMFBOdmh4UzUvUWNZSHYzR2djbVVwdjJSUGU1eXB3VWt3bEdyCm1iOXNRTFdWNW5PNzRBNWs4VkhhWjVvME5YbSt1dDgxcTdMd2hzaW40MjRyTWw1Snh4OWQxbjRvVXJOS3ZyQUkKOFV5MjlrbXE3T0Z5eGVERlVRZExiR0ZMZFVaQ3JiM01zM0ZRQVpyZDZYTUxGZm5PT1ppc2NiUDBTK2dZMHJ0aQpxclpwNVIxQkJzNWpjcEUvZ1NXaCtDQmFDaE1GVnJKdVpUZTBHTzdYQkxUWXorTmZ1aDZ2SVZYS2ZNaz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
        server: https://192.168.1.50:6443
      name: kubernetes
    contexts:
    - context:
        cluster: kubernetes
        user: kubernetes-admin
      name: kubernetes-admin@kubernetes
    current-context: kubernetes-admin@kubernetes
    kind: Config
    preferences: {}
    users:
    - name: kubernetes-admin
      user:
        client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDQWRxZ0F3SUJBZ0lJY1B1Q2JUTGtyNzR3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TURBME1EZ3dNakV3TWpKYUZ3MHlNVEEwTURnd01qRXdNamRhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXdNSTZjejg0M3lVQkFSK3AKV3QwdWZqOEhjdGJBT1V5amEzNy9HVEZJNDhGVjdEbFloTWh3akdGM016WE9GYzhMK1BlYi9DTVk2eG80VjBJSgpib1BaaUxScjRCRkJmcXpYbmNWZkg5bElDTXgxcjhwdkRCK2dUMFIzZHl5a09kaEgyZDBRTHBBenowYmdHSXJtCnFrNFlRRzdTV3RkbGIvMFlYVlRUNlU4bVR0ZzBlNWdRNGlJbUhnbXhDSCs1NzFjVXhNblprMWdkVHczK0NJTkIKaENxdGNGNDA4L0lCbjZaMTBaYUgrYjBudU5xdEZPNThhcFRtK08rU2huZzFQRnhIaERXRHljZUVtRnlWMmRBOQpZYmhhTTZKcWhRK0s2azk5Zi9xdHE1aGd0ZjVZL2h4Zjhzc05MZm0zM29PQS81MEQrTHYzclpiOXRwSVIxaHlsCjB5bnRYUUlEQVFBQm95Y3dKVEFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFDbkdhV2RLL0Y5Unl3c2pZVFIwd0ZCZ1JwcUllOGdZUGpnKwovVjYwTTBsM3JCVXNBeVRFVjU2OHkyaE9qTkpCalNoNGJPTzdGa1VZemNYeFNHSDJuMmovN3VJQytMRWVPSExDCkVXQmZ6Wm93dzR6djlpU0hUM2oyWlpncS9RZTIxcTdiWnpIMCtHUFYvSU0xa1RMc21JVHRPVzErNWM4YXpCTlcKSGJyVU9kTEV0UUI3YnI5WmZmN2lTelJzK0ZBUUVyL0pYSDhCR1p6VVRSNFlmQklNMHQ0djIxbmdoQzJYZnZpZgp5cWpmN0xySzZNYit5enRhNjZHZUlXeS9wbGxUUk5mbnhnYjgrWWxCcGpqTmJCSlg0M1cwc3IrTWVXUW5ka3UxCmlXK2ZZWW5ZRCs2UmZhb2twR1d2bmhNQnozZDBpYlByblRhbERXWHZJeFJtdDlhdmtwQT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
        client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBd01JNmN6ODQzeVVCQVIrcFd0MHVmajhIY3RiQU9VeWphMzcvR1RGSTQ4RlY3RGxZCmhNaHdqR0YzTXpYT0ZjOEwrUGViL0NNWTZ4bzRWMElKYm9QWmlMUnI0QkZCZnF6WG5jVmZIOWxJQ014MXI4cHYKREIrZ1QwUjNkeXlrT2RoSDJkMFFMcEF6ejBiZ0dJcm1xazRZUUc3U1d0ZGxiLzBZWFZUVDZVOG1UdGcwZTVnUQo0aUltSGdteENIKzU3MWNVeE1uWmsxZ2RUdzMrQ0lOQmhDcXRjRjQwOC9JQm42WjEwWmFIK2IwbnVOcXRGTzU4CmFwVG0rTytTaG5nMVBGeEhoRFdEeWNlRW1GeVYyZEE5WWJoYU02SnFoUStLNms5OWYvcXRxNWhndGY1WS9oeGYKOHNzTkxmbTMzb09BLzUwRCtMdjNyWmI5dHBJUjFoeWwweW50WFFJREFRQUJBb0lCQVFDODJZNEtlMVpzeVFSQwo1WkkydzV4WmM4Y0lhLzNJSloyMkk2WXFPRzhCTk5uSnBpVmpjajFTUyt0TThOb0g0K0lHK2hDSTVwbnpQSzBXClVFeU5TZ0JHUHYyeGVUYUJ0VFZLRGFVMHZ0d2tRcXpLbmJwT1ZtM3BPMXNRRjF5T2o2ZFZlNC92RHJpenl1eWoKSHZMK3g2UmEvRGg3WjZ5cUczMVRjMWhxckhFTHJHL2NSUGM5MmFUVVB2d256MmR1WVhHU1hUaGtrdG5Iemg2SgpiRWxkVERLcnlRcndoUW1MblowbWZIQ2NRcG1sY1NweWxmNDg2SjBka1paNjRleFJrVGlDZ0IraGNoUWxlNGJ4CjBIYW1KelIvWnVOVXoxbHFDLzEzMHBVbXV0OXpsL3RVb2R2dm4rUzR1Y282TjAxSjFObjgrd0hkM081SGNscncKM3gyVTdJWUJBb0dCQU1HbFlETlJlVk1vSmZnZk92eXF1MGoySmVkRlIzQjVBMzUvUFgxeVdFMlFvSi9WQk82VQo0TGNiVjMzd0I0VlVyRUFTeDNiZTR6V1hKUGZoSXhiUGhaVlJCTGV2Rk03ZERQMno5QUw5enBNY0twaHAwUUdhCnVieEZEMHV6ak04M0xaVFZPWnVxbElVRXVaM0prQXhnd053VjlCb2ZxeHkyL0dST0FuQUxsRlY1QW9HQkFQN1QKdGdpTTFCQ1h0MktDUXRRZFZ6Z2NxZVp1cnFIaWs2Q00vajVDRnpUNU5mOEsxTnRaTlhtTFZ0NlBlRTVZSlMwagpFVlF3d3lIUzVxZ2V3cDlGdmxuVkQ5OHNDdm9HQTNDcnUwYityc3B5UTIrZ3NCZ3JRL0ZRaTRPdG1WclV4MFB6Ckx2OG9KRlZwWGpORGs4em5zK01zZ1ZDNmhHd3RNYVEwZm04OW45SUZBb0dBTG9kUkJTT25kajZvV09VUUpGUFYKcW1OU21pNUFTeHNZcHRWbDdmV0NtQ2lQSDdoc2RmTVp4NFZ2VVZoU1Jrd2hFMGd2MnpVVS9QUnpNb2hMQ1JrVgo3Tm5KdTJUN0ovVmZRTHB6Z0NDQitVRUVUeGpsMm0vVi94SE02aENiWGRMUlJmaXgzZUJ2elVKa1l6QmlSMGNjCk1BV3FZSGlKZ2QzSjZVUUJPL0RjVkdrQ2dZQTMrZEYydDFpdC9HV3dJZVVFS3gzSm1hSklsKytNWi9UOXczcmwKdWliVzRCZFlXc3kvRWkySThXNjNuTlJVZ1ZCSlJmYThnNm1aZUhacVg3ZG92UzAvRm1wU0g1NlpwVkNFSTNVVAo5MFgxK251TnZjSnd6TEEwQmZsZmgzYTBXU0VjY0FMVzBiNkpkSWZZd3ZOb2cwMGtqZFlxSVk2TkpMQk8zYWtZClRuVVk4UUtCZ0JUMlEzVTVKMndYMkVWb0pibVYrVlBhVCt0ajFlNW9TNDdEVG8wM0phU3JMNTRGV1BsNlpIL1IKZ0FlcDRiV2c0RWphTUtNbkdPdi83THVrcGtaLzZxOXg3S2dHRGIzRWtVVHhvVFRWZFJIYk45eExPNUFCelJtOQpnWjhpM1V4dmdsYkVxRUhFWXdDUVN6OEMvK2R2Y0o2V1hHOXNla0QyVEdpakVtKzN4bGJ4Ci0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
    
    
    curl -s https://192.168.1.50:6443/api/v1/namespaces \
      --header "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6ImptcDNraWJFUEoyMmVfYkdBeVVBNnhoRnk5eTliTVNnOExndEZvSDNEV0EifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InR3aW5nYW8tdG9rZW4tdmxwN3giLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoidHdpbmdhbyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjJiOTE5OTVlLWQ2ZDItNGViYi04MGM0LWRkMTY3OTY1NjMyMSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OnR3aW5nYW8ifQ.knejqx3L372GzOtpX75qq9oIx3VCTUVn8gT1EfuJAf8PwmaPsF3QPCmbOTzlo4FIxy5oslY4bb9nEs8ibsV_fpOuXuQtUmkc6TCHrq0L8CZXDl_bEgOeWkqlSBfuNToc3W-LgKf59o8AZOuFGDfzSCA0mzUyFKNPG_KHB34s8wQrbEyO13n-2qdNHnGoqLqi8QOHtgGvoFDb3pRsLwQiOlr23H1CGKkUFcpNc7AKiyA6GffK_LsLlc-f-UH8fV0xiEmrDLWjry0BVOyvOJfB_Wiz72OwzqK2gq--QVmQSMGv6N8B1b_9gYPY31flQ0IIiz9dkcPELl5LZ__glXhWQA" \
      --cacert ca.crt
    
    
    
    curl -s https://192.168.1.50:6443/api/v1/namespaces/default/pods/ \
      --header "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6ImptcDNraWJFUEoyMmVfYkdBeVVBNnhoRnk5eTliTVNnOExndEZvSDNEV0EifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InR3aW5nYW8tdG9rZW4tdmxwN3giLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoidHdpbmdhbyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjJiOTE5OTVlLWQ2ZDItNGViYi04MGM0LWRkMTY3OTY1NjMyMSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OnR3aW5nYW8ifQ.knejqx3L372GzOtpX75qq9oIx3VCTUVn8gT1EfuJAf8PwmaPsF3QPCmbOTzlo4FIxy5oslY4bb9nEs8ibsV_fpOuXuQtUmkc6TCHrq0L8CZXDl_bEgOeWkqlSBfuNToc3W-LgKf59o8AZOuFGDfzSCA0mzUyFKNPG_KHB34s8wQrbEyO13n-2qdNHnGoqLqi8QOHtgGvoFDb3pRsLwQiOlr23H1CGKkUFcpNc7AKiyA6GffK_LsLlc-f-UH8fV0xiEmrDLWjry0BVOyvOJfB_Wiz72OwzqK2gq--QVmQSMGv6N8B1b_9gYPY31flQ0IIiz9dkcPELl5LZ__glXhWQA" \
      --cacert ca.crt
    {
      "kind": "PodList",
      "apiVersion": "v1",
      "metadata": {
        "selfLink": "/api/v1/namespaces/default/pods/",
        "resourceVersion": "319414"
      },
      "items": []
    }
    
    curl -s https://192.168.1.50:6443/api/v1/namespaces/default/services/ \
      --header "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6ImptcDNraWJFUEoyMmVfYkdBeVVBNnhoRnk5eTliTVNnOExndEZvSDNEV0EifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InR3aW5nYW8tdG9rZW4tdmxwN3giLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoidHdpbmdhbyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjJiOTE5OTVlLWQ2ZDItNGViYi04MGM0LWRkMTY3OTY1NjMyMSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OnR3aW5nYW8ifQ.knejqx3L372GzOtpX75qq9oIx3VCTUVn8gT1EfuJAf8PwmaPsF3QPCmbOTzlo4FIxy5oslY4bb9nEs8ibsV_fpOuXuQtUmkc6TCHrq0L8CZXDl_bEgOeWkqlSBfuNToc3W-LgKf59o8AZOuFGDfzSCA0mzUyFKNPG_KHB34s8wQrbEyO13n-2qdNHnGoqLqi8QOHtgGvoFDb3pRsLwQiOlr23H1CGKkUFcpNc7AKiyA6GffK_LsLlc-f-UH8fV0xiEmrDLWjry0BVOyvOJfB_Wiz72OwzqK2gq--QVmQSMGv6N8B1b_9gYPY31flQ0IIiz9dkcPELl5LZ__glXhWQA" \
      --cacert ca.crt
    {
      "kind": "ServiceList",
      "apiVersion": "v1",
      "metadata": {
        "selfLink": "/api/v1/namespaces/default/services/",
        "resourceVersion": "319946"
      },
      "items": [
        {
          "metadata": {
            "name": "kubernetes",
            "namespace": "default",
            "selfLink": "/api/v1/namespaces/default/services/kubernetes",
            "uid": "257269b1-9c19-4aba-a1b6-548048b655bf",
            "resourceVersion": "154",
            "creationTimestamp": "2020-04-08T02:10:55Z",
            "labels": {
              "component": "apiserver",
              "provider": "kubernetes"
            },
            "managedFields": [
              {
                "manager": "kube-apiserver",
                "operation": "Update",
                "apiVersion": "v1",
                "time": "2020-04-08T02:10:55Z",
                "fieldsType": "FieldsV1",
                "fieldsV1": {"f:metadata":{"f:labels":{".":{},"f:component":{},"f:provider":{}}},"f:spec":{"f:clusterIP":{},"f:ports":{".":{},"k:{\"port\":443,\"protocol\":\"TCP\"}":{".":{},"f:name":{},"f:port":{},"f:protocol":{},"f:targetPort":{}}},"f:sessionAffinity":{},"f:type":{}}}
              }
            ]
          },
          "spec": {
            "ports": [
              {
                "name": "https",
                "protocol": "TCP",
                "port": 443,
                "targetPort": 6443
              }
            ],
            "clusterIP": "10.1.0.1",
            "type": "ClusterIP",
            "sessionAffinity": "None"
          },
          "status": {
            "loadBalancer": {
    
            }
          }
        }
      ]
    }


### useraccount

    openssl genrsa -out twingao.key 2048
    
    openssl req -new -key twingao.key -out twingao.csr -subj "/CN=twingao/O=depeac"
    
    openssl x509 -req -in twingao.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out twingao.crt -days 500
    
    kubectl config set-credentials twingao --client-certificate=twingao.crt --client-key=twingao.key
    
    kubectl config set-context twingao-context --cluster=kubernetes --namespace=kube-system --user=twingao
    
    kubectl get pods --context=twingao-context
    Error from server (Forbidden): pods is forbidden: User "twingao" cannot list resource "pods" in API group "" in the namespace "kube-system"
    
    vi twingao-role.yaml
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: twingao-role
      namespace: kube-system
    rules:
    - apiGroups: ["", "extensions", "apps"]
      resources: ["deployments", "replicasets", "pods"]
      verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
    
    kubectl apply -f twingao-role.yaml
    
    vi twingao-rolebinding.yaml
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: twingao-rolebinding
      namespace: kube-system
    subjects:
    - kind: User
      name: twingao
      apiGroup: ""
    roleRef:
      kind: Role
      name: twingao-role
      apiGroup: ""
    
    kubectl apply -f twingao-rolebinding.yaml
    
    kubectl get pods --context=twingao-context
    NAME                                 READY   STATUS    RESTARTS   AGE
    coredns-7ff77c879f-2jhdx             1/1     Running   0          7h4m
    coredns-7ff77c879f-fk89q             1/1     Running   0          7h4m
    etcd-k8s-master                      1/1     Running   1          2d5h
    kube-apiserver-k8s-master            1/1     Running   0          5h15m
    kube-controller-manager-k8s-master   1/1     Running   21         2d5h
    kube-flannel-ds-amd64-8k5pg          1/1     Running   1          2d4h
    kube-flannel-ds-amd64-8nft4          1/1     Running   1          2d5h
    kube-flannel-ds-amd64-w27q7          1/1     Running   1          2d4h
    kube-proxy-b6c2q                     1/1     Running   1          2d4h
    kube-proxy-bvrhx                     1/1     Running   1          2d4h
    kube-proxy-fk7tn                     1/1     Running   1          2d5h
    kube-scheduler-k8s-master            1/1     Running   21         2d5h
    
    kubectl get pods --context=twingao-context -ndefault
    Error from server (Forbidden): pods is forbidden: User "twingao" cannot list resource "pods" in API group "" in the namespace "default"
    
    
    cat /root/.kube/config
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01EUXdPREF5TVRBeU1sb1hEVE13TURRd05qQXlNVEF5TWxvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTDZxCnZmaGtGdTdLQm5ReVBCMzNYUlovbXdIQ0lLL3FpakdMMkQrTHdJZEpFTGJDQk9QdWZPa2hFWkNWZTFVMFRoVjUKamJDbTc0cFBPNThycldFdVNFSXZ2ejlNcERDRWtoTDVhb1NDbUY2OU01WjNpMU41SmJheTUxeW05TERmR3YwcwppWVBkUS9acG9pYk4xRjl3ZzJTTU5nYjNwbjFZbS82REFWWUtqaHZFQzhyT09iWDJ5WUxBSWsxNFdiKy8zaUh1CkNEYzcrTnEzUjJLOGtkc0Q4RzdyTGVWS2krUHZHUDhPeStXUjcrelBSbEdLOWJtc1Rac3J5dUNpd09ad0hGRzUKS2o3cXhmM1pNd1hvS0dTcjdZSU16aEVrUkpTZFlTQUZGM0xLVFBoV2hlb3hwWWxWZlNuWXdJYUZpaS9CeHFHYgpuMnMza1pZUVpTYzd6eVNCRmcwQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFHd2NzVkpFNjVoUkJzS1h6WTQxVjBwQ2xXaWMKQUNZaHNuc3NkaEFaN1pqY3ZxMTVPZGIzZVh4ak1wTkw5eStsZU9vcTBPZ1hPQ3oyMGs0K21sRldFRFFxei9MbQpja3RPTHFISjd4aG9mU2xOYUROV2F1NlpsTW5DMFBOdmh4UzUvUWNZSHYzR2djbVVwdjJSUGU1eXB3VWt3bEdyCm1iOXNRTFdWNW5PNzRBNWs4VkhhWjVvME5YbSt1dDgxcTdMd2hzaW40MjRyTWw1Snh4OWQxbjRvVXJOS3ZyQUkKOFV5MjlrbXE3T0Z5eGVERlVRZExiR0ZMZFVaQ3JiM01zM0ZRQVpyZDZYTUxGZm5PT1ppc2NiUDBTK2dZMHJ0aQpxclpwNVIxQkJzNWpjcEUvZ1NXaCtDQmFDaE1GVnJKdVpUZTBHTzdYQkxUWXorTmZ1aDZ2SVZYS2ZNaz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
        server: https://192.168.1.50:6443
      name: kubernetes
    contexts:
    - context:
        cluster: kubernetes
        user: kubernetes-admin
      name: kubernetes-admin@kubernetes
    - context:
        cluster: kubernetes
        namespace: kube-system
        user: twingao
      name: twingao-context
    current-context: kubernetes-admin@kubernetes
    kind: Config
    preferences: {}
    users:
    - name: kubernetes-admin
      user:
        client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDQWRxZ0F3SUJBZ0lJY1B1Q2JUTGtyNzR3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TURBME1EZ3dNakV3TWpKYUZ3MHlNVEEwTURnd01qRXdNamRhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXdNSTZjejg0M3lVQkFSK3AKV3QwdWZqOEhjdGJBT1V5amEzNy9HVEZJNDhGVjdEbFloTWh3akdGM016WE9GYzhMK1BlYi9DTVk2eG80VjBJSgpib1BaaUxScjRCRkJmcXpYbmNWZkg5bElDTXgxcjhwdkRCK2dUMFIzZHl5a09kaEgyZDBRTHBBenowYmdHSXJtCnFrNFlRRzdTV3RkbGIvMFlYVlRUNlU4bVR0ZzBlNWdRNGlJbUhnbXhDSCs1NzFjVXhNblprMWdkVHczK0NJTkIKaENxdGNGNDA4L0lCbjZaMTBaYUgrYjBudU5xdEZPNThhcFRtK08rU2huZzFQRnhIaERXRHljZUVtRnlWMmRBOQpZYmhhTTZKcWhRK0s2azk5Zi9xdHE1aGd0ZjVZL2h4Zjhzc05MZm0zM29PQS81MEQrTHYzclpiOXRwSVIxaHlsCjB5bnRYUUlEQVFBQm95Y3dKVEFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFDbkdhV2RLL0Y5Unl3c2pZVFIwd0ZCZ1JwcUllOGdZUGpnKwovVjYwTTBsM3JCVXNBeVRFVjU2OHkyaE9qTkpCalNoNGJPTzdGa1VZemNYeFNHSDJuMmovN3VJQytMRWVPSExDCkVXQmZ6Wm93dzR6djlpU0hUM2oyWlpncS9RZTIxcTdiWnpIMCtHUFYvSU0xa1RMc21JVHRPVzErNWM4YXpCTlcKSGJyVU9kTEV0UUI3YnI5WmZmN2lTelJzK0ZBUUVyL0pYSDhCR1p6VVRSNFlmQklNMHQ0djIxbmdoQzJYZnZpZgp5cWpmN0xySzZNYit5enRhNjZHZUlXeS9wbGxUUk5mbnhnYjgrWWxCcGpqTmJCSlg0M1cwc3IrTWVXUW5ka3UxCmlXK2ZZWW5ZRCs2UmZhb2twR1d2bmhNQnozZDBpYlByblRhbERXWHZJeFJtdDlhdmtwQT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
        client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBd01JNmN6ODQzeVVCQVIrcFd0MHVmajhIY3RiQU9VeWphMzcvR1RGSTQ4RlY3RGxZCmhNaHdqR0YzTXpYT0ZjOEwrUGViL0NNWTZ4bzRWMElKYm9QWmlMUnI0QkZCZnF6WG5jVmZIOWxJQ014MXI4cHYKREIrZ1QwUjNkeXlrT2RoSDJkMFFMcEF6ejBiZ0dJcm1xazRZUUc3U1d0ZGxiLzBZWFZUVDZVOG1UdGcwZTVnUQo0aUltSGdteENIKzU3MWNVeE1uWmsxZ2RUdzMrQ0lOQmhDcXRjRjQwOC9JQm42WjEwWmFIK2IwbnVOcXRGTzU4CmFwVG0rTytTaG5nMVBGeEhoRFdEeWNlRW1GeVYyZEE5WWJoYU02SnFoUStLNms5OWYvcXRxNWhndGY1WS9oeGYKOHNzTkxmbTMzb09BLzUwRCtMdjNyWmI5dHBJUjFoeWwweW50WFFJREFRQUJBb0lCQVFDODJZNEtlMVpzeVFSQwo1WkkydzV4WmM4Y0lhLzNJSloyMkk2WXFPRzhCTk5uSnBpVmpjajFTUyt0TThOb0g0K0lHK2hDSTVwbnpQSzBXClVFeU5TZ0JHUHYyeGVUYUJ0VFZLRGFVMHZ0d2tRcXpLbmJwT1ZtM3BPMXNRRjF5T2o2ZFZlNC92RHJpenl1eWoKSHZMK3g2UmEvRGg3WjZ5cUczMVRjMWhxckhFTHJHL2NSUGM5MmFUVVB2d256MmR1WVhHU1hUaGtrdG5Iemg2SgpiRWxkVERLcnlRcndoUW1MblowbWZIQ2NRcG1sY1NweWxmNDg2SjBka1paNjRleFJrVGlDZ0IraGNoUWxlNGJ4CjBIYW1KelIvWnVOVXoxbHFDLzEzMHBVbXV0OXpsL3RVb2R2dm4rUzR1Y282TjAxSjFObjgrd0hkM081SGNscncKM3gyVTdJWUJBb0dCQU1HbFlETlJlVk1vSmZnZk92eXF1MGoySmVkRlIzQjVBMzUvUFgxeVdFMlFvSi9WQk82VQo0TGNiVjMzd0I0VlVyRUFTeDNiZTR6V1hKUGZoSXhiUGhaVlJCTGV2Rk03ZERQMno5QUw5enBNY0twaHAwUUdhCnVieEZEMHV6ak04M0xaVFZPWnVxbElVRXVaM0prQXhnd053VjlCb2ZxeHkyL0dST0FuQUxsRlY1QW9HQkFQN1QKdGdpTTFCQ1h0MktDUXRRZFZ6Z2NxZVp1cnFIaWs2Q00vajVDRnpUNU5mOEsxTnRaTlhtTFZ0NlBlRTVZSlMwagpFVlF3d3lIUzVxZ2V3cDlGdmxuVkQ5OHNDdm9HQTNDcnUwYityc3B5UTIrZ3NCZ3JRL0ZRaTRPdG1WclV4MFB6Ckx2OG9KRlZwWGpORGs4em5zK01zZ1ZDNmhHd3RNYVEwZm04OW45SUZBb0dBTG9kUkJTT25kajZvV09VUUpGUFYKcW1OU21pNUFTeHNZcHRWbDdmV0NtQ2lQSDdoc2RmTVp4NFZ2VVZoU1Jrd2hFMGd2MnpVVS9QUnpNb2hMQ1JrVgo3Tm5KdTJUN0ovVmZRTHB6Z0NDQitVRUVUeGpsMm0vVi94SE02aENiWGRMUlJmaXgzZUJ2elVKa1l6QmlSMGNjCk1BV3FZSGlKZ2QzSjZVUUJPL0RjVkdrQ2dZQTMrZEYydDFpdC9HV3dJZVVFS3gzSm1hSklsKytNWi9UOXczcmwKdWliVzRCZFlXc3kvRWkySThXNjNuTlJVZ1ZCSlJmYThnNm1aZUhacVg3ZG92UzAvRm1wU0g1NlpwVkNFSTNVVAo5MFgxK251TnZjSnd6TEEwQmZsZmgzYTBXU0VjY0FMVzBiNkpkSWZZd3ZOb2cwMGtqZFlxSVk2TkpMQk8zYWtZClRuVVk4UUtCZ0JUMlEzVTVKMndYMkVWb0pibVYrVlBhVCt0ajFlNW9TNDdEVG8wM0phU3JMNTRGV1BsNlpIL1IKZ0FlcDRiV2c0RWphTUtNbkdPdi83THVrcGtaLzZxOXg3S2dHRGIzRWtVVHhvVFRWZFJIYk45eExPNUFCelJtOQpnWjhpM1V4dmdsYkVxRUhFWXdDUVN6OEMvK2R2Y0o2V1hHOXNla0QyVEdpakVtKzN4bGJ4Ci0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
    - name: twingao
      user:
        client-certificate: /root/test/twingao.crt
        client-key: /root/test/twingao.key
    

echo 'LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDQWRxZ0F3SUJBZ0lJY1B1Q2JUTGtyNzR3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TURBME1EZ3dNakV3TWpKYUZ3MHlNVEEwTURnd01qRXdNamRhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXdNSTZjejg0M3lVQkFSK3AKV3QwdWZqOEhjdGJBT1V5amEzNy9HVEZJNDhGVjdEbFloTWh3akdGM016WE9GYzhMK1BlYi9DTVk2eG80VjBJSgpib1BaaUxScjRCRkJmcXpYbmNWZkg5bElDTXgxcjhwdkRCK2dUMFIzZHl5a09kaEgyZDBRTHBBenowYmdHSXJtCnFrNFlRRzdTV3RkbGIvMFlYVlRUNlU4bVR0ZzBlNWdRNGlJbUhnbXhDSCs1NzFjVXhNblprMWdkVHczK0NJTkIKaENxdGNGNDA4L0lCbjZaMTBaYUgrYjBudU5xdEZPNThhcFRtK08rU2huZzFQRnhIaERXRHljZUVtRnlWMmRBOQpZYmhhTTZKcWhRK0s2azk5Zi9xdHE1aGd0ZjVZL2h4Zjhzc05MZm0zM29PQS81MEQrTHYzclpiOXRwSVIxaHlsCjB5bnRYUUlEQVFBQm95Y3dKVEFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFDbkdhV2RLL0Y5Unl3c2pZVFIwd0ZCZ1JwcUllOGdZUGpnKwovVjYwTTBsM3JCVXNBeVRFVjU2OHkyaE9qTkpCalNoNGJPTzdGa1VZemNYeFNHSDJuMmovN3VJQytMRWVPSExDCkVXQmZ6Wm93dzR6djlpU0hUM2oyWlpncS9RZTIxcTdiWnpIMCtHUFYvSU0xa1RMc21JVHRPVzErNWM4YXpCTlcKSGJyVU9kTEV0UUI3YnI5WmZmN2lTelJzK0ZBUUVyL0pYSDhCR1p6VVRSNFlmQklNMHQ0djIxbmdoQzJYZnZpZgp5cWpmN0xySzZNYit5enRhNjZHZUlXeS9wbGxUUk5mbnhnYjgrWWxCcGpqTmJCSlg0M1cwc3IrTWVXUW5ka3UxCmlXK2ZZWW5ZRCs2UmZhb2twR1d2bmhNQnozZDBpYlByblRhbERXWHZJeFJtdDlhdmtwQT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=' | base64 -d > ca.pem

echo 'LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDQWRxZ0F3SUJBZ0lJY1B1Q2JUTGtyNzR3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TURBME1EZ3dNakV3TWpKYUZ3MHlNVEEwTURnd01qRXdNamRhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXdNSTZjejg0M3lVQkFSK3AKV3QwdWZqOEhjdGJBT1V5amEzNy9HVEZJNDhGVjdEbFloTWh3akdGM016WE9GYzhMK1BlYi9DTVk2eG80VjBJSgpib1BaaUxScjRCRkJmcXpYbmNWZkg5bElDTXgxcjhwdkRCK2dUMFIzZHl5a09kaEgyZDBRTHBBenowYmdHSXJtCnFrNFlRRzdTV3RkbGIvMFlYVlRUNlU4bVR0ZzBlNWdRNGlJbUhnbXhDSCs1NzFjVXhNblprMWdkVHczK0NJTkIKaENxdGNGNDA4L0lCbjZaMTBaYUgrYjBudU5xdEZPNThhcFRtK08rU2huZzFQRnhIaERXRHljZUVtRnlWMmRBOQpZYmhhTTZKcWhRK0s2azk5Zi9xdHE1aGd0ZjVZL2h4Zjhzc05MZm0zM29PQS81MEQrTHYzclpiOXRwSVIxaHlsCjB5bnRYUUlEQVFBQm95Y3dKVEFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFDbkdhV2RLL0Y5Unl3c2pZVFIwd0ZCZ1JwcUllOGdZUGpnKwovVjYwTTBsM3JCVXNBeVRFVjU2OHkyaE9qTkpCalNoNGJPTzdGa1VZemNYeFNHSDJuMmovN3VJQytMRWVPSExDCkVXQmZ6Wm93dzR6djlpU0hUM2oyWlpncS9RZTIxcTdiWnpIMCtHUFYvSU0xa1RMc21JVHRPVzErNWM4YXpCTlcKSGJyVU9kTEV0UUI3YnI5WmZmN2lTelJzK0ZBUUVyL0pYSDhCR1p6VVRSNFlmQklNMHQ0djIxbmdoQzJYZnZpZgp5cWpmN0xySzZNYit5enRhNjZHZUlXeS9wbGxUUk5mbnhnYjgrWWxCcGpqTmJCSlg0M1cwc3IrTWVXUW5ka3UxCmlXK2ZZWW5ZRCs2UmZhb2twR1d2bmhNQnozZDBpYlByblRhbERXWHZJeFJtdDlhdmtwQT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=' | base64 -d > admin-client.pem

echo $clientkey | base64 -d > ./client-key.pem




    
curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem https://172.21.0.15:6443/api/v1/pods
    
    curl $APISERVER/api/v1/pods --cert /home/ca.cert --key /home/ca.key --user system:admin
    
    
curl -i https://192.168.1.50:6443/api/v1/pods \
  --cert /root/test/twingao.crt \
  --key /root/test/twingao.key \
  --cacert /root/test/ca.pem \
  --user twingao
    
