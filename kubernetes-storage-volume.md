# Kubernetes存储卷介绍-emptyDir/hostPath/NFS/configMap

## 存储卷介绍

一个容器的对文件系统的写入都是发生在文件系统的可写层的，一旦该容器运行结束，所有写入数据都会被丢弃。在K8S集群之中，Pod会在各个节点之间漂移，如何保障Pod的数据持久和不同节点数据的共享。Kubernetes提出了存储卷Volume的概念，Kubernetes存储卷主要解决了依次递增的几个问题：
- 当运行的容器崩溃时，kubelet会重新启动该容器，但容器会以干净状态被重新启动。容器崩溃之前写入的文件将会被丢失。
- 当一个Pod中同时运行多个容器时，这些容器之间需要共享文件。
- 在k8s中，由于Pod分布在各个不同的节点之上，并不能实现不同节点之间持久性数据的共享，并且在节点故障时，可能会导致数据的永久性丢失。

Kubernetes存储卷拥有明确的生命周期，与所在的Pod的生命周期相同。因此Kubernetes存储卷独立于任何容器，所以数据在Pod重启的过程中还会保留，当然如果这个Pod被删除了，那么这些数据也会被删除。

Kubernetes 支持的卷类型

| Type | Type | Type | Type  | Type | Type |
| :------| :------ | :------ | :------ | :------| :------ | 
| awsElasticBlockStore | azureDisk | azureFile | cephfs | cinder | <strong>configMap</strong> |
| csi | downwardAPI | <strong>emptyDir</strong> | fc (fibre channel) | flexVolume | flocker |
| gcePersistentDisk | glusterfs | <strong>hostPath</strong> | iscsi | local | <strong>nfs</strong> |
| <strong>persistentVolumeClaim</strong> | projected | portworxVolume |quobyte | rbd | scaleIO | secret |
| storageos | vsphereVolume |

## emptyDir

emptyDir类型的存储卷创建于Pod被调度到宿主机上的时候，同一个Pod内的多个容器能读写emptyDir中的同一个文件，一旦这个Pod销毁或者漂移开该宿主机，emptyDir中的数据就会被永久删除。emptyDir类型的存储卷主要用作临时空间或者不同容器之间的数据共享。容器的crashing事件并不会导致emptyDir中的数据被删除。

emptyDir支持内存作为存储资源，emptyDir.medium设为Memory即可，但如果遇到node节点重启，emptyDir中的数据也会全部丢失。

以下演示如何在Pod中不同容器之间共享数据。

    vi emptydir-pod.yaml
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: emptydir-pod
    spec:
      #定义emptyDir存储卷，名称my-emptydir-vol
      volumes:
        - name: my-emptydir-vol
          emptyDir: {}
      containers:
        - name: busybox-1
          image: busybox:latest
          command:
            - "sleep"
            - "3600"
          #将my-emptydir-vol存储卷Mount到容器中
          volumeMounts:
            - name: my-emptydir-vol
              #此路径为容器中目录
              mountPath: /data-1
        - name: busybox-2
          image: busybox:latest
          command:
            - "sleep"
            - "3600"
          #将my-emptydir-vol存储卷Mount到容器中
          volumeMounts:
            - name: my-emptydir-vol
              mountPath: /data-2
    
    kubectl apply -f emptydir-pod.yaml

进入容器busybox-1，存在/data-1目录

    kubectl exec -it emptydir-pod -c busybox-1 sh

    ls
    bin     data-1  dev     etc     home    proc    root    sys     tmp     usr     var
    
在容器busybox-1向共享目录中/data-1创建一个文件

    cd data-1
    echo "hello world" > hello.txt
    exit

容器busybox-2的共享目录中/data-2确实能够访问容器busybox-1创建的文件，注意busybox-1:/data-1 = busybox-2:/data-2

    kubectl exec -it emptydir-pod -c busybox-2 sh
    cat /data-2/hello.txt
    hello world
    exit

我们看看emptyDir存储卷在宿主机的位置。在k8s-node1节点，在Pod临时目录下存在my-emptydir-vol目录，容器创建的文件就存放在该目录中。Pod的临时目录会在Pod销毁时删除，其中my-emptydir-vol也会被附带删除。

    kubectl get pod -o yaml | grep nodeName
        nodeName: k8s-node1
    kubectl get pod -o yaml | grep uid
        uid: e70923f9-156a-4fde-98be-e77f3f65fbd9

    #在k8s-node1节点
    tree /var/lib/kubelet/pods/e70923f9-156a-4fde-98be-e77f3f65fbd9/
    /var/lib/kubelet/pods/e70923f9-156a-4fde-98be-e77f3f65fbd9/
    ├── containers
    │   ├── busybox-1
    │   │   ├── 753e3ae2
    │   │   └── 96560e76
    │   └── busybox-2
    │       ├── 1a8b5849
    │       └── a6a696a2
    ├── etc-hosts
    ├── plugins
    │   └── kubernetes.io~empty-dir
    │       ├── my-emptydir-vol
    │       │   └── ready
    │       └── wrapped_default-token-wtqgl
    │           └── ready
    └── volumes
        ├── kubernetes.io~empty-dir
        │   └── my-emptydir-vol
        │       └── hello.txt
        └── kubernetes.io~secret
            └── default-token-wtqgl
                ├── ca.crt -> ..data/ca.crt
                ├── namespace -> ..data/namespace
                └── token -> ..data/token
    
    cat /var/lib/kubelet/pods/e70923f9-156a-4fde-98be-e77f3f65fbd9/volumes/kubernetes.io~empty-dir/my-emptydir-vol/hello.txt
    hello world

删除Pod，查看Pod临时目录。发现已经没有Pod临时目录e70923f9-156a-4fde-98be-e77f3f65fbd9了。

    kubectl delete -f emptydir-pod.yaml

    #在k8s-node1节点
    ls /var/lib/kubelet/pods/

## hostPath

hostPath存储卷为pod挂载宿主机上的目录或文件，使得容器可以使用宿主机的文件系统进行存储。缺点是不提供Pod的亲和性，即hostPath映射的目录在node1，当Pod可能被调度到node2，导致原来的在node1的数据不存在，Pod一直无法启动起来。

hostPath存储卷类型

| 值 | 行为 |
| :------| :------ | 
| 空 | 空字符串（默认）用于向后兼容，这意味着在安装hostPath卷之前不会执行任何检查。 |
| DirectoryOrCreate | 如果给定路径中不存在任何内容，则将根据需要创建一个空目录，权限设置为0755，与Kubelet具有相同的组和所有权。 |
| Directory | 目录必须存在于给定路径中 |
| FileOrCreate | 如果给定路径中不存在任何内容，则会根据需要创建一个空文件，权限设置为0644，与Kubelet具有相同的组和所有权。 |
| File | 文件必须存在于给定路径中 |
| Socket | UNIX套接字必须存在于给定路径中 |
| CharDevice | 字符设备必须存在于给定路径中 |
| BlockDevice | 块设备必须存在于给定路径中 |

以下演示以下hostPath存储卷的使用。可以先看一下各个node节点是否存在/data目录。
    
    vi hostpath-pod.yaml
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: hostpath-pod
    spec:
      volumes:
        - name: my-hostpath-vol
          hostPath:
        #为宿主机的共享目录
            path: /data/hostpath-1
        #如果宿主机目录不存在，则创建
            type: DirectoryOrCreate
      containers:
        - name: busybox
          image: busybox:latest
          #将my-hostpath-vol存储卷Mount到/data目录，/data目录为容器中目录
          volumeMounts:
            - mountPath: /data
              name: my-hostpath-vol
          command:
            - "/bin/sh"
            - "-c"
            - "echo 'hello world' > /data/hello.txt; sleep 3600" 
    
    kubectl apply -f hostpath-pod.yaml

    kubectl get pod -o wide
    NAME           READY   STATUS    RESTARTS   AGE    IP            NODE        NOMINATED NODE   READINESS GATES
    hostpath-pod   1/1     Running   0          20s    10.244.2.23   k8s-node2   <none>           <none>

在k8s-node2节点查看，已经创建了一个共享目录/data/hostpath-1，并且其中确实存在容器新创建的文件。

    cd /data/hostpath-1
    cat hello.txt
    hello world

删除Pod，可以发现k8s-node2节点中的共享目录和文件仍然还存在。

    kubectl delete -f hostpath-pod.yaml

    cat /data/hostpath-1/hello.txt
    hello world

## NFS

emptyDir在Pod销毁时会自动删除，hostPath无法随着Pod在node节点之间漂移。和NFS不同，NFS服务器独立提供共享文件系统，Pod直接Mount共享目录，Pod漂移到另一个node节点时，会重新自动Mount共享目录，数据不会丢失，可以在Pod和node之间共享。

NFS服务器的搭建请参见[NFS v4的安装和使用-CentOS 7](../nfs/nfs-v4-centos-installation-introduction.md)。并创建一个新的共享目录。

    mkdir /data/vol -p
    chmod 777 /data/vol
    
    vim /etc/exports
    /data/vol                     192.168.1.0/24(sync,rw,no_root_squash)

    exportfs -r

    showmount -e nfs
    Export list for nfs:
    /data/vol    192.168.1.0/24

在Kubernetes的所有Worker节点安装NFS客户端，并测试是否能够连接NFS服务器。

    yum install -y nfs-utils

    showmount -e 192.168.1.80
    Export list for 192.168.1.80:
    /data/vol      192.168.1.0/24

以下演示一下NFS存储卷的使用。创建两个Pod，分别部署到两个不同的Node节点，并同时向同一个文件中写入数据。

    vi nfs-pod-1.yaml    
    apiVersion: v1
    kind: Pod
    metadata:
      name: nfs-pod-1
    spec:
      nodeName: k8s-node1
      volumes:
        - name: my-nfs-vol
          nfs:
            #NFS服务器上创建的共享目录
            path: /data/vol
            #NFS服务器IP地址
            server: 192.168.1.80
      containers:
        - name: busybox
          image: busybox:latest
          volumeMounts:
            #关联存储卷名称
            - name: my-nfs-vol
              #容器内目录，NFS的共享目录就是Mount到该目录的
              mountPath: /data
          command:
          - "/bin/sh"
          - "-c"
          - "while true; do echo 'nfs-pod-1' >> /data/hello.txt; sleep 2; done"

    vi nfs-pod-2.yaml    
    apiVersion: v1
    kind: Pod
    metadata:
      name: nfs-pod-2
    spec:
      nodeName: k8s-node2
      volumes:
        - name: my-nfs-vol
          nfs:
            #NFS服务器上创建的共享目录
            path: /data/vol
            #NFS服务器IP地址
            server: 192.168.1.80
      containers:
        - name: busybox
          image: busybox:latest
          volumeMounts:
            #关联存储卷名称
            - name: my-nfs-vol
              #容器内目录，NFS的共享目录就是Mount到该目录的
              mountPath: /data
          command:
          - "/bin/sh"
          - "-c"
          - "while true; do echo 'nfs-pod-2' >> /data/hello.txt; sleep 2; done"

    kubectl apply -f nfs-pod-1.yaml
    kubectl apply -f nfs-pod-2.yaml

在NFS服务器查看结果。可以看出两个不同的Pod在不同的node节点可以同时Mount同一个共享目录，并同时向一个文件中写入数据。

    tail -f /data/vol/hello.txt
    nfs-pod-1
    nfs-pod-1
    nfs-pod-1
    nfs-pod-2
    nfs-pod-1
    nfs-pod-2
    nfs-pod-1
    nfs-pod-2
    nfs-pod-1
    nfs-pod-2
    nfs-pod-1
    nfs-pod-2

## configMap

configMap用于保存配置数据的键值对，可以用来保存单个属性，也可以用来保存配置文件。可以通过环境变量的方式使用configMap，也可以通过存储卷的方式使用configMap，下面我们演示如何通过存储卷使用configMap。

    vi my-configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: my-configmap
    data:
      age: "12"
      name: |+
        firstName=Zhang
        lastName=San
    
    vi configmap-pod.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: configmap-pod
    spec:
      volumes:
        - name: my-configmap-vol
          configMap:
            name: my-configmap
      containers:
        - name: busybox
          image: busybox:latest
          volumeMounts:
            - name: my-configmap-vol
              mountPath: /data
          command:
            - "sleep"
            - "3600"
    
    kubectl apply -f my-configmap.yaml
    kubectl apply -f configmap-pod.yaml


进入Pod，可以看出创建了一个/data目录，其中包含两个文件age和name，分别对应configMap中的两个键值对，键名为文件名，键值为文件内容。

    kubectl exec -it configmap-pod sh
    ls
    bin   data  dev   etc   home  proc  root  sys   tmp   usr   var
    cd /data
    ls
    age   name
    cat age
    12
    cat name
    firstName=Zhang
    lastName=San
    
    exit

configMap作为存储卷使用时可以热更新。将my-configmap的键name的键值增加一个属性，并重新应用，可以发现容器中的文件内容发生了变化。热更新间隔时间缺省为小于1分钟，也就是configMap的内容更新后在1分钟之内容器中的文件内容会随之更新。

    vi my-configmap-new.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: my-configmap
    data:
      age: "12"
      name: |+
        firstName=Zhang
        lastName=San
        userName=zhangsan
    
    kubectl apply -f configmap-pod-new.yaml
    
    kubectl exec -it configmap-pod sh
    cat /data/name
    firstName=Zhang
    lastName=San
    userName=zhangsan

其它类型的存储就不详细介绍了。