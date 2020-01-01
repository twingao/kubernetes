# CentOS 7下Kubernetes 1.16.4安装

系统环境。

	cat /etc/redhat-release
	CentOS Linux release 7.7.1908 (Core)
	
	uname -a
	Linux k8s-master 3.10.0-1062.9.1.el7.x86_64 #1 SMP Fri Dec 6 15:49:49 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
	
	vi /etc/hosts
	192.168.1.55 k8s-master
	192.168.1.56 k8s-node1
	192.168.1.57 k8s-node2
	192.168.1.58 k8s-node3

关闭防火墙和安全设置。

	systemctl stop firewalld
	systemctl disable firewalld
	
	vi /etc/fstab
	#/dev/mapper/centos-swap swap                    swap    defaults        0 0
	
	vi /etc/selinux/config
	SELINUX=disabled
	
	#重启生效
	reboot

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
	可安装的软件包
	 * updates: mirrors.huaweicloud.com
	Loading mirror speeds from cached hostfile
	 * extras: mirror.bit.edu.cn
	docker-ce.x86_64            3:19.03.5-3.el7                     docker-ce-stable
	docker-ce.x86_64            3:19.03.4-3.el7                     docker-ce-stable
	docker-ce.x86_64            3:19.03.3-3.el7                     docker-ce-stable
	docker-ce.x86_64            3:19.03.2-3.el7                     docker-ce-stable
	docker-ce.x86_64            3:19.03.1-3.el7                     docker-ce-stable
	docker-ce.x86_64            3:19.03.0-3.el7                     docker-ce-stable
	docker-ce.x86_64            3:18.09.9-3.el7                     docker-ce-stable
	docker-ce.x86_64            3:18.09.8-3.el7                     docker-ce-stable
	docker-ce.x86_64            3:18.09.7-3.el7                     docker-ce-stable
	docker-ce.x86_64            3:18.09.6-3.el7                     docker-ce-stable
	docker-ce.x86_64            3:18.09.5-3.el7                     docker-ce-stable
	docker-ce.x86_64            3:18.09.4-3.el7                     docker-ce-stable
	docker-ce.x86_64            3:18.09.3-3.el7                     docker-ce-stable
	docker-ce.x86_64            3:18.09.2-3.el7                     docker-ce-stable
	......
	 * base: mirrors.tuna.tsinghua.edu.cn
	
	yum install -y docker-ce-19.03.5-3.el7
	
	docker version
	Client: Docker Engine - Community
	 Version:           19.03.5
	 API version:       1.40
	 Go version:        go1.12.12
	 Git commit:        633a0ea
	 Built:             Wed Nov 13 07:25:41 2019
	 OS/Arch:           linux/amd64
	 Experimental:      false
	
	Server: Docker Engine - Community
	 Engine:
	  Version:          19.03.5
	  API version:      1.40 (minimum version 1.12)
	  Go version:       go1.12.12
	  Git commit:       633a0ea
	  Built:            Wed Nov 13 07:24:18 2019
	  OS/Arch:          linux/amd64
	  Experimental:     false
	 containerd:
	  Version:          1.2.10
	  GitCommit:        b34a5c8af56e510852c35414db4c1f4fa6172339
	 runc:
	  Version:          1.0.0-rc8+dev
	  GitCommit:        3e425f80a8c931f88e6d94a8c831b9d5aa481657
	 docker-init:
	  Version:          0.18.0
	  GitCommit:        fec3683
	
	systemctl start docker
	systemctl enable docker

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

安装kubeadmin、kubectl和kubelet服务。

	yum list -y kubeadm --showduplicates
	已加载插件：fastestmirror
	Loading mirror speeds from cached hostfile
	 * base: mirror.bit.edu.cn
	 * extras: mirror.bit.edu.cn
	 * updates: mirror.jdcloud.com
	可安装的软件包
	kubeadm.x86_64                                                          1.6.0-0                                                             kubernetes
	kubeadm.x86_64                                                          1.6.1-0                                                             kubernetes
	kubeadm.x86_64                                                          1.6.2-0                                                             kubernetes
	kubeadm.x86_64                                                          1.6.3-0                                                             kubernetes
	......
	kubeadm.x86_64                                                          1.13.0-0                                                            kubernetes
	kubeadm.x86_64                                                          1.13.1-0                                                            kubernetes
	kubeadm.x86_64                                                          1.13.2-0                                                            kubernetes
	kubeadm.x86_64                                                          1.13.3-0                                                            kubernetes
	kubeadm.x86_64                                                          1.13.4-0                                                            kubernetes
	kubeadm.x86_64                                                          1.13.5-0                                                            kubernetes
	kubeadm.x86_64                                                          1.13.6-0                                                            kubernetes
	kubeadm.x86_64                                                          1.13.7-0                                                            kubernetes
	kubeadm.x86_64                                                          1.13.8-0                                                            kubernetes
	kubeadm.x86_64                                                          1.13.9-0                                                            kubernetes
	kubeadm.x86_64                                                          1.13.10-0                                                           kubernetes
	kubeadm.x86_64                                                          1.13.11-0                                                           kubernetes
	kubeadm.x86_64                                                          1.13.12-0                                                           kubernetes
	kubeadm.x86_64                                                          1.14.0-0                                                            kubernetes
	kubeadm.x86_64                                                          1.14.1-0                                                            kubernetes
	kubeadm.x86_64                                                          1.14.2-0                                                            kubernetes
	kubeadm.x86_64                                                          1.14.3-0                                                            kubernetes
	kubeadm.x86_64                                                          1.14.4-0                                                            kubernetes
	kubeadm.x86_64                                                          1.14.5-0                                                            kubernetes
	kubeadm.x86_64                                                          1.14.6-0                                                            kubernetes
	kubeadm.x86_64                                                          1.14.7-0                                                            kubernetes
	kubeadm.x86_64                                                          1.14.8-0                                                            kubernetes
	kubeadm.x86_64                                                          1.14.9-0                                                            kubernetes
	kubeadm.x86_64                                                          1.14.10-0                                                           kubernetes
	kubeadm.x86_64                                                          1.15.0-0                                                            kubernetes
	kubeadm.x86_64                                                          1.15.1-0                                                            kubernetes
	kubeadm.x86_64                                                          1.15.2-0                                                            kubernetes
	kubeadm.x86_64                                                          1.15.3-0                                                            kubernetes
	kubeadm.x86_64                                                          1.15.4-0                                                            kubernetes
	kubeadm.x86_64                                                          1.15.5-0                                                            kubernetes
	kubeadm.x86_64                                                          1.15.6-0                                                            kubernetes
	kubeadm.x86_64                                                          1.15.7-0                                                            kubernetes
	kubeadm.x86_64                                                          1.16.0-0                                                            kubernetes
	kubeadm.x86_64                                                          1.16.1-0                                                            kubernetes
	kubeadm.x86_64                                                          1.16.2-0                                                            kubernetes
	kubeadm.x86_64                                                          1.16.3-0                                                            kubernetes
	kubeadm.x86_64                                                          1.16.4-0                                                            kubernetes
	kubeadm.x86_64                                                          1.17.0-0                                                            kubernetes
	
	yum install -y kubeadm-1.16.4-0 kubectl-1.16.4-0 kubelet-1.16.4-0
	
	kubeadm version
	kubeadm version: &version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.4", GitCommit:"224be7bdce5a9dd0c2fd0d46b83865648e2fe0ba", GitTreeState:"clean", BuildDate:"2019-12-11T12:44:45Z", GoVersion:"go1.12.12", Compiler:"gc", Platform:"linux/amd64"}
	
	systemctl start kubelet
	systemctl enable kubelet

查看Kubernetes所需要的镜像版本号。并提前下载好所需的七个镜像。

	kubeadm config images pull --image-repository=registry.aliyuncs.com/google_containers
	
	W1220 21:38:53.193429   18919 version.go:101] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://dl.k8s.io/release/stable-1.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
	W1220 21:38:53.193913   18919 version.go:102] falling back to the local client version: v1.16.4
	[config/images] Pulled registry.aliyuncs.com/google_containers/kube-apiserver:v1.16.4
	[config/images] Pulled registry.aliyuncs.com/google_containers/kube-controller-manager:v1.16.4
	[config/images] Pulled registry.aliyuncs.com/google_containers/kube-scheduler:v1.16.4
	[config/images] Pulled registry.aliyuncs.com/google_containers/kube-proxy:v1.16.4
	[config/images] Pulled registry.aliyuncs.com/google_containers/pause:3.1
	[config/images] Pulled registry.aliyuncs.com/google_containers/etcd:3.3.15-0
	[config/images] Pulled registry.aliyuncs.com/google_containers/coredns:1.6.2
	
	docker images
	REPOSITORY                                                        TAG                 IMAGE ID            CREATED             SIZE
	registry.aliyuncs.com/google_containers/kube-apiserver            v1.16.4             3722a80984a0        9 days ago          217MB
	registry.aliyuncs.com/google_containers/kube-controller-manager   v1.16.4             fb4cca6b4e4c        9 days ago          163MB
	registry.aliyuncs.com/google_containers/kube-proxy                v1.16.4             091df896d78f        9 days ago          86.1MB
	registry.aliyuncs.com/google_containers/kube-scheduler            v1.16.4             2984964036c8        9 days ago          87.3MB
	registry.aliyuncs.com/google_containers/etcd                      3.3.15-0            b2756210eeab        3 months ago        247MB
	registry.aliyuncs.com/google_containers/coredns                   1.6.2               bf261d157914        4 months ago        44.1MB
	registry.aliyuncs.com/google_containers/pause                     3.1                 da86e6ba6ca1        24 months ago       742kB

> 以上所有的命令需要在k8s-master、node1、node2和node3四个节点都操作。以下初始化操作只需要在k8s-master节点操作。

初始化Master节点。

	# –-pod-network-cidr：用于指定Pod的网络范围
	# –-service-cidr：用于指定service的网络范围；
	# --image-repository: 镜像仓库的地址，和提前下载的镜像仓库应该对应上。
	
	kubeadm init --kubernetes-version=v1.16.4 \
	--pod-network-cidr=10.244.0.0/16 \
	--service-cidr=10.1.0.0/16 \
	--image-repository=registry.aliyuncs.com/google_containers
	
	[init] Using Kubernetes version: v1.16.4
	[preflight] Running pre-flight checks
	        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
	        [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 19.03.5. Latest validated version: 18.09
	[preflight] Pulling images required for setting up a Kubernetes cluster
	[preflight] This might take a minute or two, depending on the speed of your internet connection
	[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
	[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
	[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
	[kubelet-start] Activating the kubelet service
	[certs] Using certificateDir folder "/etc/kubernetes/pki"
	[certs] Generating "ca" certificate and key
	[certs] Generating "apiserver" certificate and key
	[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.1.0.1 192.168.1.55]
	[certs] Generating "apiserver-kubelet-client" certificate and key
	[certs] Generating "front-proxy-ca" certificate and key
	[certs] Generating "front-proxy-client" certificate and key
	[certs] Generating "etcd/ca" certificate and key
	[certs] Generating "etcd/server" certificate and key
	[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.1.55 127.0.0.1 ::1]
	[certs] Generating "etcd/peer" certificate and key
	[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.1.55 127.0.0.1 ::1]
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
	[control-plane] Creating static Pod manifest for "kube-scheduler"
	[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
	[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
	[kubelet-check] Initial timeout of 40s passed.
	[apiclient] All control plane components are healthy after 50.510284 seconds
	[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
	[kubelet] Creating a ConfigMap "kubelet-config-1.16" in namespace kube-system with the configuration for the kubelets in the cluster
	[upload-certs] Skipping phase. Please see --upload-certs
	[mark-control-plane] Marking the node k8s-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
	[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
	[bootstrap-token] Using token: nll94a.w2g4xm386y5tjbjx
	[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
	[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
	[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
	[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
	[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
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
	
	kubeadm join 192.168.1.55:6443 --token nll94a.w2g4xm386y5tjbjx \
	    --discovery-token-ca-cert-hash sha256:29c411866c0c0c7f2647f0c45b188951d535f8ac29465542a5538c351e473a58

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
	k8s-master   NotReady   master   6m31s   v1.16.4
	
	kubectl get all -n kube-system
	NAME                                     READY   STATUS    RESTARTS   AGE
	pod/coredns-58cc8c89f4-88nbr             0/1     Pending   0          5m37s
	pod/coredns-58cc8c89f4-h4glp             0/1     Pending   0          5m37s
	pod/etcd-k8s-master                      1/1     Running   0          4m47s
	pod/kube-apiserver-k8s-master            1/1     Running   0          4m45s
	pod/kube-controller-manager-k8s-master   1/1     Running   0          4m52s
	pod/kube-proxy-tmbpd                     1/1     Running   0          5m37s
	pod/kube-scheduler-k8s-master            1/1     Running   0          4m43s
	
	NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
	service/kube-dns   ClusterIP   10.1.0.10    <none>        53/UDP,53/TCP,9153/TCP   5m52s
	
	NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
	daemonset.apps/kube-proxy   1         1         1       1            1           beta.kubernetes.io/os=linux   5m52s
	
	NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
	deployment.apps/coredns   0/2     2            0           5m52s
	
	NAME                                 DESIRED   CURRENT   READY   AGE
	replicaset.apps/coredns-58cc8c89f4   2         2         0       5m37s

部署flannel网络。

	mkdir k8s
	cd k8s
	
	#该地址可能无法访问。
	wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
	
	#直接下载代码仓库 https://github.com/coreos/flannel，下载为flannel-master.zip文件，解压文件，
	#将flannel-master\Documentation\kube-flannel.yml上传到k8s-master节点。
	
	# 可以提前将镜像拉下来。四个节点都操作。
	docker pull quay.io/coreos/flannel:v0.11.0-amd64
	
	kubectl apply -f kube-flannel.yml
	
	kubectl get nodes
	NAME         STATUS   ROLES    AGE   VERSION
	k8s-master   Ready    master   35m   v1.16.4
	
	kubectl get all -n kube-system
	NAME                                     READY   STATUS    RESTARTS   AGE
	pod/coredns-58cc8c89f4-88nbr             1/1     Running   0          34m
	pod/coredns-58cc8c89f4-h4glp             1/1     Running   0          34m
	pod/etcd-k8s-master                      1/1     Running   1          33m
	pod/kube-apiserver-k8s-master            1/1     Running   1          33m
	pod/kube-controller-manager-k8s-master   1/1     Running   1          33m
	pod/kube-flannel-ds-amd64-5prwj          1/1     Running   0          100s
	pod/kube-proxy-tmbpd                     1/1     Running   1          34m
	pod/kube-scheduler-k8s-master            1/1     Running   1          33m
	
	NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
	service/kube-dns   ClusterIP   10.1.0.10    <none>        53/UDP,53/TCP,9153/TCP   34m
	
	NAME                                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
	daemonset.apps/kube-flannel-ds-amd64     1         1         1       1            1           <none>                        100s
	daemonset.apps/kube-flannel-ds-arm       0         0         0       0            0           <none>                        99s
	daemonset.apps/kube-flannel-ds-arm64     0         0         0       0            0           <none>                        100s
	daemonset.apps/kube-flannel-ds-ppc64le   0         0         0       0            0           <none>                        99s
	daemonset.apps/kube-flannel-ds-s390x     0         0         0       0            0           <none>                        99s
	daemonset.apps/kube-proxy                1         1         1       1            1           beta.kubernetes.io/os=linux   34m
	
	NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
	deployment.apps/coredns   2/2     2            2           34m
	
	NAME                                 DESIRED   CURRENT   READY   AGE
	replicaset.apps/coredns-58cc8c89f4   2         2         2       34m

> 以上命令只在k8s-master节点执行。

Join node节点到Kubernetes集群中。分别在k8s-node1、k8s-node2和k8s-node3节点执行如下join命令，并执行配置命令。

	kubeadm join 192.168.1.55:6443 --token nll94a.w2g4xm386y5tjbjx \
	    --discovery-token-ca-cert-hash sha256:29c411866c0c0c7f2647f0c45b188951d535f8ac29465542a5538c351e473a58
	
	[preflight] Running pre-flight checks
	        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
	        [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 19.03.5. Latest validated version: 18.09
	[preflight] Reading configuration from the cluster...
	[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
	[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.16" ConfigMap in the kube-system namespace
	[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
	[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
	[kubelet-start] Activating the kubelet service
	[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
	
	This node has joined the cluster:
	* Certificate signing request was sent to apiserver and a response was received.
	* The Kubelet was informed of the new secure connection details.
	
	Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
	
	mkdir -p $HOME/.kube
	scp 192.168.1.55:/root/.kube/config $HOME/.kube/config

查看kubernetes状态。

	kubectl get nodes
	NAME         STATUS   ROLES    AGE     VERSION
	k8s-master   Ready    master   46m     v1.16.4
	k8s-node1    Ready    <none>   6m2s    v1.16.4
	k8s-node2    Ready    <none>   5m52s   v1.16.4
	k8s-node3    Ready    <none>   5m52s   v1.16.4
	
	kubectl get all -n kube-system
	NAME                                     READY   STATUS    RESTARTS   AGE
	pod/coredns-58cc8c89f4-88nbr             1/1     Running   0          46m
	pod/coredns-58cc8c89f4-h4glp             1/1     Running   0          46m
	pod/etcd-k8s-master                      1/1     Running   1          46m
	pod/kube-apiserver-k8s-master            1/1     Running   1          46m
	pod/kube-controller-manager-k8s-master   1/1     Running   1          46m
	pod/kube-flannel-ds-amd64-5dkkn          1/1     Running   0          6m25s
	pod/kube-flannel-ds-amd64-5prwj          1/1     Running   0          13m
	pod/kube-flannel-ds-amd64-xrpft          1/1     Running   0          6m35s
	pod/kube-proxy-7966v                     1/1     Running   0          6m25s
	pod/kube-proxy-9tvwr                     1/1     Running   0          6m35s
	pod/kube-proxy-tmbpd                     1/1     Running   1          46m
	pod/kube-scheduler-k8s-master            1/1     Running   1          45m
	
	NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
	service/kube-dns   ClusterIP   10.1.0.10    <none>        53/UDP,53/TCP,9153/TCP   47m
	
	NAME                                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
	daemonset.apps/kube-flannel-ds-amd64     3         3         3       3            3           <none>                        13m
	daemonset.apps/kube-flannel-ds-arm       0         0         0       0            0           <none>                        13m
	daemonset.apps/kube-flannel-ds-arm64     0         0         0       0            0           <none>                        13m
	daemonset.apps/kube-flannel-ds-ppc64le   0         0         0       0            0           <none>                        13m
	daemonset.apps/kube-flannel-ds-s390x     0         0         0       0            0           <none>                        13m
	daemonset.apps/kube-proxy                3         3         3       3            3           beta.kubernetes.io/os=linux   47m
	
	NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
	deployment.apps/coredns   2/2     2            2           47m
	
	NAME                                 DESIRED   CURRENT   READY   AGE
	replicaset.apps/coredns-58cc8c89f4   2         2         2       46m

完毕。