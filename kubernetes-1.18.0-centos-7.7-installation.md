# CentOS 7.7下Kubernetes 1.18.0安装

系统环境。

    cat /etc/redhat-release
    CentOS Linux release 7.7.1908 (Core)
    
    uname -a
    Linux k8s-master 3.10.0-1062.18.1.el7.x86_64 #1 SMP Tue Mar 17 23:49:17 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
    
    vi /etc/hosts
    192.168.1.50 k8s-master
    192.168.1.51 k8s-node1
    192.168.1.52 k8s-node2
    192.168.1.53 k8s-node3

关闭防火墙和安全设置。

    systemctl stop firewalld
    systemctl disable firewalld
    
    vi /etc/fstab
    #/dev/mapper/centos-swap swap                    swap    defaults        0 0
    
    vi /etc/selinux/config
    SELINUX=disabled
    
    #重启生效
    reboot

配置免密码登录。在每个节点都执行ssh-keygen产生公私钥对，都选缺省值，一路回车即可。

    ssh-keygen

用ssh-copy-id将本节点的公钥复制到其它节点，如k8s-master节点，需要将公钥复制到k8s-node1、k8s-node2和k8s-node3三个节点，其它节点都要类似操作。

    ssh-copy-id k8s-node1
    ssh-copy-id k8s-node2
    ssh-copy-id k8s-node3

修改内核参数。

    cat <<EOF> /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    
    modprobe br_netfilter
    sysctl --system
    
    * Applying /usr/lib/sysctl.d/00-system.conf ...
    net.bridge.bridge-nf-call-ip6tables = 0
    net.bridge.bridge-nf-call-iptables = 0
    net.bridge.bridge-nf-call-arptables = 0
    * Applying /usr/lib/sysctl.d/10-default-yama-scope.conf ...
    kernel.yama.ptrace_scope = 0
    * Applying /usr/lib/sysctl.d/50-default.conf ...
    kernel.sysrq = 16
    kernel.core_uses_pid = 1
    net.ipv4.conf.default.rp_filter = 1
    net.ipv4.conf.all.rp_filter = 1
    net.ipv4.conf.default.accept_source_route = 0
    net.ipv4.conf.all.accept_source_route = 0
    net.ipv4.conf.default.promote_secondaries = 1
    net.ipv4.conf.all.promote_secondaries = 1
    fs.protected_hardlinks = 1
    fs.protected_symlinks = 1
    * Applying /etc/sysctl.d/99-sysctl.conf ...
    * Applying /etc/sysctl.d/k8s.conf ...
    #注意需要有以下两行
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    * Applying /etc/sysctl.conf ...

安装Docker。

    yum install -y yum-utils device-mapper-persistent-data lvm2
    
    yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    或者
    #yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    
    yum list docker-ce --showduplicates | sort -r
    已加载插件：fastestmirror
    已安装的软件包
    可安装的软件包
     * updates: mirror.bit.edu.cn
    Loading mirror speeds from cached hostfile
     * extras: mirrors.huaweicloud.com
    docker-ce.x86_64            3:19.03.8-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:19.03.8-3.el7                    @docker-ce-stable
    docker-ce.x86_64            3:19.03.7-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:19.03.6-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:19.03.5-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:19.03.4-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:19.03.3-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:19.03.2-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:19.03.1-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:19.03.0-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.9-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.8-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.7-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.6-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.5-3.el7                    docker-ce-stable
    ......
    
    yum install -y docker-ce-19.03.8-3.el7

    systemctl start docker
    systemctl enable docker

    docker version
    Client: Docker Engine - Community
     Version:           19.03.8
     API version:       1.40
     Go version:        go1.12.17
     Git commit:        afacb8b
     Built:             Wed Mar 11 01:27:04 2020
     OS/Arch:           linux/amd64
     Experimental:      false
    
    Server: Docker Engine - Community
     Engine:
      Version:          19.03.8
      API version:      1.40 (minimum version 1.12)
      Go version:       go1.12.17
      Git commit:       afacb8b
      Built:            Wed Mar 11 01:25:42 2020
      OS/Arch:          linux/amd64
      Experimental:     false
     containerd:
      Version:          1.2.13
      GitCommit:        7ad184331fa3e55e52b890ea95e65ba581ae3429
     runc:
      Version:          1.0.0-rc10
      GitCommit:        dc9208a3303feef5b3839f4323d9beb36df0a9dd
     docker-init:
      Version:          0.18.0
      GitCommit:        fec3683

设置Kubernetes的yum源。

    cat <<EOF> /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
    EOF

或

    cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=http://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=http://packages.cloud.google.com/yum/doc/yum-key.gpg http://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    EOF

安装kubeadmin、kubectl和kubelet服务。

    yum list -y kubeadm --showduplicates
    已加载插件：fastestmirror
    Loading mirror speeds from cached hostfile
     * base: mirror.bit.edu.cn
     * extras: mirrors.huaweicloud.com
     * updates: mirror.bit.edu.cn
    kubernetes/signature                                                                                   |  454 B  00:00:00
    从 https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg 检索密钥
    导入 GPG key 0xA7317B0F:
     用户ID     : "Google Cloud Packages Automatic Signing Key <gc-team@google.com>"
     指纹       : d0bc 747f d8ca f711 7500 d6fa 3746 c208 a731 7b0f
     来自       : https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
    从 https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg 检索密钥
    kubernetes/signature                                                                                   | 1.4 kB  00:00:00 !!!
    kubernetes/primary                                                                                     |  66 kB  00:00:00
    kubernetes                                                                                                            481/481
    可安装的软件包
    kubeadm.x86_64                                              1.6.0-0                                                 kubernetes
    kubeadm.x86_64                                              1.6.1-0                                                 kubernetes
    kubeadm.x86_64                                              1.6.2-0                                                 kubernetes
    kubeadm.x86_64                                              1.6.3-0                                                 kubernetes
    kubeadm.x86_64                                              1.6.4-0                                                 kubernetes
    kubeadm.x86_64                                              1.6.5-0                                                 kubernetes
    kubeadm.x86_64                                              1.6.6-0                                                 kubernetes
    kubeadm.x86_64                                              1.6.7-0                                                 kubernetes
    kubeadm.x86_64                                              1.6.8-0                                                 kubernetes
    kubeadm.x86_64                                              1.6.9-0                                                 kubernetes
    kubeadm.x86_64                                              1.6.10-0                                                kubernetes
    kubeadm.x86_64                                              1.6.11-0                                                kubernetes
    kubeadm.x86_64                                              1.6.12-0                                                kubernetes
    kubeadm.x86_64                                              1.6.13-0                                                kubernetes
    ......
    kubeadm.x86_64                                              1.12.0-0                                                kubernetes
    kubeadm.x86_64                                              1.12.1-0                                                kubernetes
    kubeadm.x86_64                                              1.12.2-0                                                kubernetes
    kubeadm.x86_64                                              1.12.3-0                                                kubernetes
    kubeadm.x86_64                                              1.12.4-0                                                kubernetes
    kubeadm.x86_64                                              1.12.5-0                                                kubernetes
    kubeadm.x86_64                                              1.12.6-0                                                kubernetes
    kubeadm.x86_64                                              1.12.7-0                                                kubernetes
    kubeadm.x86_64                                              1.12.8-0                                                kubernetes
    kubeadm.x86_64                                              1.12.9-0                                                kubernetes
    kubeadm.x86_64                                              1.12.10-0                                               kubernetes
    ......
    kubeadm.x86_64                                              1.16.0-0                                                kubernetes
    kubeadm.x86_64                                              1.16.1-0                                                kubernetes
    kubeadm.x86_64                                              1.16.2-0                                                kubernetes
    kubeadm.x86_64                                              1.16.3-0                                                kubernetes
    kubeadm.x86_64                                              1.16.4-0                                                kubernetes
    kubeadm.x86_64                                              1.16.5-0                                                kubernetes
    kubeadm.x86_64                                              1.16.6-0                                                kubernetes
    kubeadm.x86_64                                              1.16.7-0                                                kubernetes
    kubeadm.x86_64                                              1.16.8-0                                                kubernetes
    kubeadm.x86_64                                              1.17.0-0                                                kubernetes
    kubeadm.x86_64                                              1.17.1-0                                                kubernetes
    kubeadm.x86_64                                              1.17.2-0                                                kubernetes
    kubeadm.x86_64                                              1.17.3-0                                                kubernetes
    kubeadm.x86_64                                              1.17.4-0                                                kubernetes
    kubeadm.x86_64                                              1.18.0-0                                                kubernetes

    yum install -y kubeadm-1.18.0-0 kubectl-1.18.0-0 kubelet-1.18.0-0
    
    kubeadm version
    kubeadm version: &version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.0", GitCommit:"9e991415386e4cf155a24b1da15becaa390438d8", GitTreeState:"clean", BuildDate:"2020-03-25T14:56:30Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"linux/amd64"}
    
    systemctl start kubelet
    systemctl enable kubelet

查看Kubernetes所需要的镜像版本号，由于k8s.gcr.io需要科学，可以使用国内镜像。

    kubeadm config images list
    W0408 09:52:39.571992    1651 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
    k8s.gcr.io/kube-apiserver:v1.18.0
    k8s.gcr.io/kube-controller-manager:v1.18.0
    k8s.gcr.io/kube-scheduler:v1.18.0
    k8s.gcr.io/kube-proxy:v1.18.0
    k8s.gcr.io/pause:3.2
    k8s.gcr.io/etcd:3.4.3-0
    k8s.gcr.io/coredns:1.6.7

可以在国内镜像仓库提前下载好所需的七个镜像，不加--image-repository即可从官方仓库下载镜像。

    kubeadm config images pull --image-repository=registry.aliyuncs.com/google_containers
    W0408 09:53:39.571992    1651 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
    [config/images] Pulled registry.aliyuncs.com/google_containers/kube-apiserver:v1.18.0
    [config/images] Pulled registry.aliyuncs.com/google_containers/kube-controller-manager:v1.18.0
    [config/images] Pulled registry.aliyuncs.com/google_containers/kube-scheduler:v1.18.0
    [config/images] Pulled registry.aliyuncs.com/google_containers/kube-proxy:v1.18.0
    [config/images] Pulled registry.aliyuncs.com/google_containers/pause:3.2
    [config/images] Pulled registry.aliyuncs.com/google_containers/etcd:3.4.3-0
    [config/images] Pulled registry.aliyuncs.com/google_containers/coredns:1.6.7

    docker images
    REPOSITORY                                                        TAG                 IMAGE ID            CREATED             SIZE
    registry.aliyuncs.com/google_containers/kube-proxy                v1.18.0             43940c34f24f        13 days ago         117MB
    registry.aliyuncs.com/google_containers/kube-apiserver            v1.18.0             74060cea7f70        13 days ago         173MB
    registry.aliyuncs.com/google_containers/kube-controller-manager   v1.18.0             d3e55153f52f        13 days ago         162MB
    registry.aliyuncs.com/google_containers/kube-scheduler            v1.18.0             a31f78c7c8ce        13 days ago         95.3MB
    registry.aliyuncs.com/google_containers/pause                     3.2                 80d28bedfe5d        7 weeks ago         683kB
    registry.aliyuncs.com/google_containers/coredns                   1.6.7               67da37a9a360        2 months ago        43.8MB
    registry.aliyuncs.com/google_containers/etcd                      3.4.3-0             303ce5db0e90        5 months ago        288MB

> 以上所有的命令需要在k8s-master、node1、node2和node3四个节点都操作。以下初始化操作只需要在k8s-master节点操作。

初始化Master节点。

    # –-pod-network-cidr：用于指定Pod的网络范围
    # –-service-cidr：用于指定service的网络范围；
    # --image-repository: 镜像仓库的地址，和提前下载的镜像仓库应该对应上。
    
    kubeadm init --kubernetes-version=v1.18.0 \
      --pod-network-cidr=10.244.0.0/16 \
      --service-cidr=10.1.0.0/16 \
      --image-repository=registry.aliyuncs.com/google_containers
    W0408 10:10:19.704855    9534 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
    [init] Using Kubernetes version: v1.18.0
    [preflight] Running pre-flight checks
            [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
    [preflight] Pulling images required for setting up a Kubernetes cluster
    [preflight] This might take a minute or two, depending on the speed of your internet connection
    [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
    [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [kubelet-start] Starting the kubelet
    [certs] Using certificateDir folder "/etc/kubernetes/pki"
    [certs] Generating "ca" certificate and key
    [certs] Generating "apiserver" certificate and key
    [certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.1.0.1 192.168.1.50]
    [certs] Generating "apiserver-kubelet-client" certificate and key
    [certs] Generating "front-proxy-ca" certificate and key
    [certs] Generating "front-proxy-client" certificate and key
    [certs] Generating "etcd/ca" certificate and key
    [certs] Generating "etcd/server" certificate and key
    [certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.1.50 127.0.0.1 ::1]
    [certs] Generating "etcd/peer" certificate and key
    [certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.1.50 127.0.0.1 ::1]
    [certs] Generating "etcd/healthcheck-client" certificate and key
    [certs] Generating "apiserver-etcd-client" certificate and key
    [certs] Generating "sa" key and public key
    [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
    [kubeconfig] Writing "admin.conf" kubeconfig file
    [kubeconfig] Writing "kubelet.conf" kubeconfig file
    [kubeconfig] Writing "controller-manager.conf" kubeconfig file
    [kubeconfig] Writing "scheduler.conf" kubeconfig file
    [control-plane] Using manifest folder "/etc/kubernetes/manifests"
    [control-plane] Creating static Pod manifest for "kube-apiserver"
    [control-plane] Creating static Pod manifest for "kube-controller-manager"
    W0408 10:10:29.102901    9534 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
    [control-plane] Creating static Pod manifest for "kube-scheduler"
    W0408 10:10:29.104505    9534 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
    [etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
    [wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
    [apiclient] All control plane components are healthy after 26.007875 seconds
    [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
    [upload-certs] Skipping phase. Please see --upload-certs
    [mark-control-plane] Marking the node k8s-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
    [mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
    [bootstrap-token] Using token: 4oxcgj.1dqz97nbu4pcf84l
    [bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
    [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
    [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
    [kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy
    
    Your Kubernetes control-plane has initialized successfully!
    
    To start using your cluster, you need to run the following as a regular user:
    
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      https://kubernetes.io/docs/concepts/cluster-administration/addons/
    
    Then you can join any number of worker nodes by running the following on each as root:
    
    kubeadm join 192.168.1.50:6443 --token 4oxcgj.1dqz97nbu4pcf84l \
        --discovery-token-ca-cert-hash sha256:2445a08ab9e210e9d3f82949ae16472d47abbc188a2b28e4d6470b02d5ddce3a

初始化完成后，需要按照提示执行以下命令。注意最后的join命令后续会使用到，需要记录。

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

修改Service的nodePort缺省取值范围。

    vi /etc/kubernetes/manifests/kube-apiserver.yaml
    - --service-node-port-range=1-39999
    
    systemctl daemon-reload
    systemctl restart kubelet

查看Kubernetes状态。

    kubectl get nodes
    NAME         STATUS     ROLES    AGE     VERSION
    k8s-master   NotReady   master   4m45s   v1.18.0
    
    kubectl get all -n kube-system
    NAME                                     READY   STATUS    RESTARTS   AGE
    pod/coredns-7ff77c879f-8flq5             0/1     Pending   0          5m53s
    pod/coredns-7ff77c879f-xth7n             0/1     Pending   0          5m54s
    pod/etcd-k8s-master                      1/1     Running   0          6m3s
    pod/kube-apiserver-k8s-master            1/1     Running   0          19s
    pod/kube-controller-manager-k8s-master   1/1     Running   1          6m2s
    pod/kube-proxy-fk7tn                     1/1     Running   0          5m54s
    pod/kube-scheduler-k8s-master            1/1     Running   1          6m2s
    
    NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
    service/kube-dns   ClusterIP   10.1.0.10    <none>        53/UDP,53/TCP,9153/TCP   6m11s
    
    NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
    daemonset.apps/kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   6m11s
    
    NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/coredns   0/2     2            0           6m11s
    
    NAME                                 DESIRED   CURRENT   READY   AGE
    replicaset.apps/coredns-7ff77c879f   2         2         0       5m54s

部署flannel网络。

    wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    
以上地址可能无法访问，可以直接下载代码仓库[https://github.com/coreos/flannel](https://github.com/coreos/flannel)，解压flannel-master.zip文件，将`flannel-master\Documentation\kube-flannel.yml`上传到k8s-master节点。

建议将kube-flannel.yml中的kube-flannel-ds-arm64、kube-flannel-ds-arm、kube-flannel-ds-ppc64le和kube-flannel-ds-s390x这4个daemonset删除，只保留kube-flannel-ds-amd64，因为一般的实验环境都是amd64架构的。

可以提前将镜像拉下来，四个节点都操作。

    docker pull quay.io/coreos/flannel:v0.12.0-amd64

    kubectl apply -f kube-flannel.yml
    podsecuritypolicy.policy/psp.flannel.unprivileged created
    clusterrole.rbac.authorization.k8s.io/flannel created
    clusterrolebinding.rbac.authorization.k8s.io/flannel created
    serviceaccount/flannel created
    configmap/kube-flannel-cfg created
    daemonset.apps/kube-flannel-ds-amd64 created
    
    kubectl get nodes
    NAME         STATUS   ROLES    AGE   VERSION
    k8s-master   Ready    master   14m   v1.18.0
        
    kubectl get all -n kube-system
    NAME                                     READY   STATUS    RESTARTS   AGE
    pod/coredns-7ff77c879f-8flq5             1/1     Running   0          23m
    pod/coredns-7ff77c879f-xth7n             1/1     Running   0          23m
    pod/etcd-k8s-master                      1/1     Running   0          24m
    pod/kube-apiserver-k8s-master            1/1     Running   0          18m
    pod/kube-controller-manager-k8s-master   1/1     Running   1          24m
    pod/kube-flannel-ds-amd64-8nft4          1/1     Running   0          13m
    pod/kube-proxy-fk7tn                     1/1     Running   0          23m
    pod/kube-scheduler-k8s-master            1/1     Running   1          24m
    
    NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
    service/kube-dns   ClusterIP   10.1.0.10    <none>        53/UDP,53/TCP,9153/TCP   24m
    
    NAME                                   DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
    daemonset.apps/kube-flannel-ds-amd64   1         1         1       1            1           <none>                   13m
    daemonset.apps/kube-proxy              1         1         1       1            1           kubernetes.io/os=linux   24m
    
    NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/coredns   2/2     2            2           24m
    
    NAME                                 DESIRED   CURRENT   READY   AGE
    replicaset.apps/coredns-7ff77c879f   2         2         2       23m

> 以上命令只在k8s-master节点执行。

Join node节点到Kubernetes集群中。分别在k8s-node1、k8s-node2和k8s-node3节点执行如下join命令，并执行配置命令。

    kubeadm join 192.168.1.50:6443 --token 4oxcgj.1dqz97nbu4pcf84l \
        --discovery-token-ca-cert-hash sha256:2445a08ab9e210e9d3f82949ae16472d47abbc188a2b28e4d6470b02d5ddce3a    
    W0408 10:42:33.434915   10437 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
    [preflight] Running pre-flight checks
            [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
    [preflight] Reading configuration from the cluster...
    [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
    [kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.18" ConfigMap in the kube-system namespace
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
    [kubelet-start] Starting the kubelet
    [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
    
    This node has joined the cluster:
    * Certificate signing request was sent to apiserver and a response was received.
    * The Kubelet was informed of the new secure connection details.
    
    Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

为了在node节点上执行kubectl命令，可以如下操作。

    mkdir -p $HOME/.kube
    scp 192.168.1.50:/root/.kube/config $HOME/.kube/config

查看kubernetes状态。

    kubectl get nodes
    NAME         STATUS   ROLES    AGE     VERSION
    k8s-master   Ready    master   34m     v1.18.0
    k8s-node1    Ready    <none>   2m47s   v1.18.0
    k8s-node2    Ready    <none>   2m45s   v1.18.0
    k8s-node3    Ready    <none>   2m43s   v1.18.0
    
    kubectl get all -n kube-system
    NAME                                     READY   STATUS    RESTARTS   AGE
    pod/coredns-7ff77c879f-8flq5             1/1     Running   0          35m
    pod/coredns-7ff77c879f-xth7n             1/1     Running   0          35m
    pod/etcd-k8s-master                      1/1     Running   0          35m
    pod/kube-apiserver-k8s-master            1/1     Running   0          29m
    pod/kube-controller-manager-k8s-master   1/1     Running   1          35m
    pod/kube-flannel-ds-amd64-8k5pg          1/1     Running   0          3m41s
    pod/kube-flannel-ds-amd64-8nft4          1/1     Running   0          24m
    pod/kube-flannel-ds-amd64-w27q7          1/1     Running   0          3m39s
    pod/kube-proxy-b6c2q                     1/1     Running   0          3m39s
    pod/kube-proxy-bvrhx                     1/1     Running   0          3m41s
    pod/kube-proxy-fk7tn                     1/1     Running   0          35m
    pod/kube-scheduler-k8s-master            1/1     Running   1          35m
    
    NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
    service/kube-dns   ClusterIP   10.1.0.10    <none>        53/UDP,53/TCP,9153/TCP   35m
    
    NAME                                   DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
    daemonset.apps/kube-flannel-ds-amd64   3         3         3       3            3           <none>                   24m
    daemonset.apps/kube-proxy              3         3         3       3            3           kubernetes.io/os=linux   35m
    
    NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/coredns   2/2     2            2           35m
    
    NAME                                 DESIRED   CURRENT   READY   AGE
    replicaset.apps/coredns-7ff77c879f   2         2         2       35m

完毕。