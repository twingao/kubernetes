# 通过etcdctl查询Kubernetes中etcd数据

先确定Kubernetes的etcd版本。

    docker images | grep etcd
    registry.aliyuncs.com/google_containers/etcd                      3.3.15-0            b2756210eeab        6 months ago        247MB

下载etcd二级制版本文件[https://github.com/etcd-io/etcd/releases](https://github.com/etcd-io/etcd/releases)，其中包含了etcdctl执行文件。找到相应release版本下载。或者按照以下脚本下载安装。

    vi download-etcd.sh
    #!/bin/bash
    ETCD_VER=v3.3.15
    ETCD_DIR=/tmp/etcd-download
    DOWNLOAD_URL=https://github.com/etcd-io/etcd/releases/download
    
    # Download
    mkdir ${ETCD_DIR} -p
    cd ${ETCD_DIR}
    wget ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz 
    tar -xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz
    
    # install
    cd etcd-${ETCD_VER}-linux-amd64
    cp etcdctl /usr/local/bin/

    chmod 755 download-etcd.sh
    ./download-etcd.sh

由于Kubernetes使用etcd v3版本的API，而且etcd默认使用tls，所以先配置几个环境变量。

    vi /etc/profile
    export ETCDCTL_API=3
    export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
    export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/peer.crt
    export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/peer.key

    source /etc/profile

获取所有的key列表。--prefix表示查找所有以/registry为前缀的key，--keys-only=true表示只给出key，不给出value。

    etcdctl get /registry --prefix --keys-only=true | grep '/registry'
    /registry/apiextensions.k8s.io/customresourcedefinitions/kongconsumers.configuration.konghq.com
    /registry/apiextensions.k8s.io/customresourcedefinitions/kongcredentials.configuration.konghq.com
    /registry/apiextensions.k8s.io/customresourcedefinitions/kongingresses.configuration.konghq.com
    /registry/apiextensions.k8s.io/customresourcedefinitions/kongplugins.configuration.konghq.com
    /registry/apiregistration.k8s.io/apiservices/v1.
    /registry/apiregistration.k8s.io/apiservices/v1.admissionregistration.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1.apiextensions.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1.apps
    /registry/apiregistration.k8s.io/apiservices/v1.authentication.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1.authorization.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1.autoscaling
    /registry/apiregistration.k8s.io/apiservices/v1.batch
    /registry/apiregistration.k8s.io/apiservices/v1.configuration.konghq.com
    /registry/apiregistration.k8s.io/apiservices/v1.coordination.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1.networking.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1.rbac.authorization.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1.scheduling.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1.storage.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1beta1.admissionregistration.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1beta1.apiextensions.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1beta1.authentication.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1beta1.authorization.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1beta1.batch
    /registry/apiregistration.k8s.io/apiservices/v1beta1.certificates.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1beta1.coordination.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1beta1.events.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1beta1.extensions
    /registry/apiregistration.k8s.io/apiservices/v1beta1.networking.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1beta1.node.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1beta1.policy
    /registry/apiregistration.k8s.io/apiservices/v1beta1.rbac.authorization.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1beta1.scheduling.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v1beta1.storage.k8s.io
    /registry/apiregistration.k8s.io/apiservices/v2beta1.autoscaling
    /registry/apiregistration.k8s.io/apiservices/v2beta2.autoscaling
    /registry/clusterrolebindings/cluster-admin
    /registry/clusterrolebindings/flannel
    /registry/clusterrolebindings/gateway-kong
    /registry/clusterrolebindings/kubeadm:kubelet-bootstrap
    /registry/clusterrolebindings/kubeadm:node-autoapprove-bootstrap
    /registry/clusterrolebindings/kubeadm:node-autoapprove-certificate-rotation
    /registry/clusterrolebindings/kubeadm:node-proxier
    /registry/clusterrolebindings/system:basic-user
    /registry/clusterrolebindings/system:controller:attachdetach-controller
    /registry/clusterrolebindings/system:controller:certificate-controller
    /registry/clusterrolebindings/system:controller:clusterrole-aggregation-controller
    /registry/clusterrolebindings/system:controller:cronjob-controller
    /registry/clusterrolebindings/system:controller:daemon-set-controller
    /registry/clusterrolebindings/system:controller:deployment-controller
    /registry/clusterrolebindings/system:controller:disruption-controller
    /registry/clusterrolebindings/system:controller:endpoint-controller
    /registry/clusterrolebindings/system:controller:expand-controller
    /registry/clusterrolebindings/system:controller:generic-garbage-collector
    /registry/clusterrolebindings/system:controller:horizontal-pod-autoscaler
    /registry/clusterrolebindings/system:controller:job-controller
    /registry/clusterrolebindings/system:controller:namespace-controller
    /registry/clusterrolebindings/system:controller:node-controller
    /registry/clusterrolebindings/system:controller:persistent-volume-binder
    /registry/clusterrolebindings/system:controller:pod-garbage-collector
    /registry/clusterrolebindings/system:controller:pv-protection-controller
    /registry/clusterrolebindings/system:controller:pvc-protection-controller
    /registry/clusterrolebindings/system:controller:replicaset-controller
    /registry/clusterrolebindings/system:controller:replication-controller
    /registry/clusterrolebindings/system:controller:resourcequota-controller
    /registry/clusterrolebindings/system:controller:route-controller
    /registry/clusterrolebindings/system:controller:service-account-controller
    /registry/clusterrolebindings/system:controller:service-controller
    /registry/clusterrolebindings/system:controller:statefulset-controller
    /registry/clusterrolebindings/system:controller:ttl-controller
    /registry/clusterrolebindings/system:coredns
    /registry/clusterrolebindings/system:discovery
    /registry/clusterrolebindings/system:kube-controller-manager
    /registry/clusterrolebindings/system:kube-dns
    /registry/clusterrolebindings/system:kube-scheduler
    /registry/clusterrolebindings/system:node
    /registry/clusterrolebindings/system:node-proxier
    /registry/clusterrolebindings/system:public-info-viewer
    /registry/clusterrolebindings/system:volume-scheduler
    /registry/clusterroles/admin
    /registry/clusterroles/cluster-admin
    /registry/clusterroles/edit
    /registry/clusterroles/flannel
    /registry/clusterroles/gateway-kong
    /registry/clusterroles/system:aggregate-to-admin
    /registry/clusterroles/system:aggregate-to-edit
    /registry/clusterroles/system:aggregate-to-view
    /registry/clusterroles/system:auth-delegator
    /registry/clusterroles/system:basic-user
    /registry/clusterroles/system:certificates.k8s.io:certificatesigningrequests:nodeclient
    /registry/clusterroles/system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
    /registry/clusterroles/system:controller:attachdetach-controller
    /registry/clusterroles/system:controller:certificate-controller
    /registry/clusterroles/system:controller:clusterrole-aggregation-controller
    /registry/clusterroles/system:controller:cronjob-controller
    /registry/clusterroles/system:controller:daemon-set-controller
    /registry/clusterroles/system:controller:deployment-controller
    /registry/clusterroles/system:controller:disruption-controller
    /registry/clusterroles/system:controller:endpoint-controller
    /registry/clusterroles/system:controller:expand-controller
    /registry/clusterroles/system:controller:generic-garbage-collector
    /registry/clusterroles/system:controller:horizontal-pod-autoscaler
    /registry/clusterroles/system:controller:job-controller
    /registry/clusterroles/system:controller:namespace-controller
    /registry/clusterroles/system:controller:node-controller
    /registry/clusterroles/system:controller:persistent-volume-binder
    /registry/clusterroles/system:controller:pod-garbage-collector
    /registry/clusterroles/system:controller:pv-protection-controller
    /registry/clusterroles/system:controller:pvc-protection-controller
    /registry/clusterroles/system:controller:replicaset-controller
    /registry/clusterroles/system:controller:replication-controller
    /registry/clusterroles/system:controller:resourcequota-controller
    /registry/clusterroles/system:controller:route-controller
    /registry/clusterroles/system:controller:service-account-controller
    /registry/clusterroles/system:controller:service-controller
    /registry/clusterroles/system:controller:statefulset-controller
    /registry/clusterroles/system:controller:ttl-controller
    /registry/clusterroles/system:coredns
    /registry/clusterroles/system:csi-external-attacher
    /registry/clusterroles/system:csi-external-provisioner
    /registry/clusterroles/system:discovery
    /registry/clusterroles/system:heapster
    /registry/clusterroles/system:kube-aggregator
    /registry/clusterroles/system:kube-controller-manager
    /registry/clusterroles/system:kube-dns
    /registry/clusterroles/system:kube-scheduler
    /registry/clusterroles/system:kubelet-api-admin
    /registry/clusterroles/system:node
    /registry/clusterroles/system:node-bootstrapper
    /registry/clusterroles/system:node-problem-detector
    /registry/clusterroles/system:node-proxier
    /registry/clusterroles/system:persistent-volume-provisioner
    /registry/clusterroles/system:public-info-viewer
    /registry/clusterroles/system:volume-scheduler
    /registry/clusterroles/view
    /registry/configmaps/default/gateway-kong-default-custom-server-blocks
    /registry/configmaps/default/kong-ingress-controller-leader-kong-kong
    /registry/configmaps/kube-public/cluster-info
    /registry/configmaps/kube-system/coredns
    /registry/configmaps/kube-system/extension-apiserver-authentication
    /registry/configmaps/kube-system/kube-flannel-cfg
    /registry/configmaps/kube-system/kube-proxy
    /registry/configmaps/kube-system/kubeadm-config
    /registry/configmaps/kube-system/kubelet-config-1.16
    /registry/controllerrevisions/kube-system/kube-flannel-ds-amd64-67f65bfbc7
    /registry/controllerrevisions/kube-system/kube-flannel-ds-arm-74f7486b59
    /registry/controllerrevisions/kube-system/kube-flannel-ds-arm64-575fdc5885
    /registry/controllerrevisions/kube-system/kube-flannel-ds-ppc64le-84596b9cb9
    /registry/controllerrevisions/kube-system/kube-flannel-ds-s390x-7f96755bd4
    /registry/controllerrevisions/kube-system/kube-proxy-74fb9dc88
    /registry/daemonsets/kube-system/kube-flannel-ds-amd64
    /registry/daemonsets/kube-system/kube-flannel-ds-arm
    /registry/daemonsets/kube-system/kube-flannel-ds-arm64
    /registry/daemonsets/kube-system/kube-flannel-ds-ppc64le
    /registry/daemonsets/kube-system/kube-flannel-ds-s390x
    /registry/daemonsets/kube-system/kube-proxy
    /registry/deployments/default/gateway-kong
    /registry/deployments/kube-system/coredns
    /registry/leases/kube-node-lease/k8s-master
    /registry/leases/kube-node-lease/k8s-node1
    /registry/leases/kube-node-lease/k8s-node2
    /registry/masterleases/192.168.1.55
    /registry/minions/k8s-master
    /registry/minions/k8s-node1
    /registry/minions/k8s-node2
    /registry/namespaces/default
    /registry/namespaces/kube-node-lease
    /registry/namespaces/kube-public
    /registry/namespaces/kube-system
    /registry/pods/default/gateway-kong-67fb7768ff-2zlxq
    /registry/pods/default/gateway-kong-67fb7768ff-sgfzk
    /registry/pods/kube-system/coredns-58cc8c89f4-88nbr
    /registry/pods/kube-system/coredns-58cc8c89f4-h4glp
    /registry/pods/kube-system/etcd-k8s-master
    /registry/pods/kube-system/kube-apiserver-k8s-master
    /registry/pods/kube-system/kube-controller-manager-k8s-master
    /registry/pods/kube-system/kube-flannel-ds-amd64-5dkkn
    /registry/pods/kube-system/kube-flannel-ds-amd64-5prwj
    /registry/pods/kube-system/kube-flannel-ds-amd64-xrpft
    /registry/pods/kube-system/kube-proxy-7966v
    /registry/pods/kube-system/kube-proxy-9tvwr
    /registry/pods/kube-system/kube-proxy-tmbpd
    /registry/pods/kube-system/kube-scheduler-k8s-master
    /registry/podsecuritypolicy/psp.flannel.unprivileged
    /registry/priorityclasses/system-cluster-critical
    /registry/priorityclasses/system-node-critical
    /registry/ranges/serviceips
    /registry/ranges/servicenodeports
    /registry/replicasets/default/gateway-kong-67fb7768ff
    /registry/replicasets/kube-system/coredns-58cc8c89f4
    /registry/rolebindings/default/gateway-kong
    /registry/rolebindings/kube-public/kubeadm:bootstrap-signer-clusterinfo
    /registry/rolebindings/kube-public/system:controller:bootstrap-signer
    /registry/rolebindings/kube-system/kube-proxy
    /registry/rolebindings/kube-system/kubeadm:kubelet-config-1.16
    /registry/rolebindings/kube-system/kubeadm:nodes-kubeadm-config
    /registry/rolebindings/kube-system/system::extension-apiserver-authentication-reader
    /registry/rolebindings/kube-system/system::leader-locking-kube-controller-manager
    /registry/rolebindings/kube-system/system::leader-locking-kube-scheduler
    /registry/rolebindings/kube-system/system:controller:bootstrap-signer
    /registry/rolebindings/kube-system/system:controller:cloud-provider
    /registry/rolebindings/kube-system/system:controller:token-cleaner
    /registry/roles/default/gateway-kong
    /registry/roles/kube-public/kubeadm:bootstrap-signer-clusterinfo
    /registry/roles/kube-public/system:controller:bootstrap-signer
    /registry/roles/kube-system/extension-apiserver-authentication-reader
    /registry/roles/kube-system/kube-proxy
    /registry/roles/kube-system/kubeadm:kubelet-config-1.16
    /registry/roles/kube-system/kubeadm:nodes-kubeadm-config
    /registry/roles/kube-system/system::leader-locking-kube-controller-manager
    /registry/roles/kube-system/system::leader-locking-kube-scheduler
    /registry/roles/kube-system/system:controller:bootstrap-signer
    /registry/roles/kube-system/system:controller:cloud-provider
    /registry/roles/kube-system/system:controller:token-cleaner
    /registry/secrets/default/default-token-wtqgl
    /registry/secrets/default/gateway-kong-token-z5wv8
    /registry/secrets/default/sh.helm.release.v1.gateway.v1
    /registry/secrets/kube-node-lease/default-token-rxvqw
    /registry/secrets/kube-public/default-token-mb8jm
    /registry/secrets/kube-system/attachdetach-controller-token-96r5d
    /registry/secrets/kube-system/bootstrap-signer-token-6vn9q
    /registry/secrets/kube-system/certificate-controller-token-hc688
    /registry/secrets/kube-system/clusterrole-aggregation-controller-token-bkj96
    /registry/secrets/kube-system/coredns-token-q4gdw
    /registry/secrets/kube-system/cronjob-controller-token-6d769
    /registry/secrets/kube-system/daemon-set-controller-token-jsd2x
    /registry/secrets/kube-system/default-token-tqqtk
    /registry/secrets/kube-system/deployment-controller-token-2zdf4
    /registry/secrets/kube-system/disruption-controller-token-6pn4c
    /registry/secrets/kube-system/endpoint-controller-token-tvjc2
    /registry/secrets/kube-system/expand-controller-token-96dkj
    /registry/secrets/kube-system/flannel-token-g5rbh
    /registry/secrets/kube-system/generic-garbage-collector-token-nbj8m
    /registry/secrets/kube-system/horizontal-pod-autoscaler-token-l5h66
    /registry/secrets/kube-system/job-controller-token-rf2hc
    /registry/secrets/kube-system/kube-proxy-token-mqzr4
    /registry/secrets/kube-system/namespace-controller-token-bxw9m
    /registry/secrets/kube-system/node-controller-token-9ls5w
    /registry/secrets/kube-system/persistent-volume-binder-token-wxxk7
    /registry/secrets/kube-system/pod-garbage-collector-token-9lcb7
    /registry/secrets/kube-system/pv-protection-controller-token-wpzf6
    /registry/secrets/kube-system/pvc-protection-controller-token-986dk
    /registry/secrets/kube-system/replicaset-controller-token-w9p6t
    /registry/secrets/kube-system/replication-controller-token-t5sq9
    /registry/secrets/kube-system/resourcequota-controller-token-lbwht
    /registry/secrets/kube-system/service-account-controller-token-fsvb9
    /registry/secrets/kube-system/service-controller-token-vw9zl
    /registry/secrets/kube-system/statefulset-controller-token-lkj86
    /registry/secrets/kube-system/token-cleaner-token-rpjrc
    /registry/secrets/kube-system/ttl-controller-token-sfnqj
    /registry/serviceaccounts/default/default
    /registry/serviceaccounts/default/gateway-kong
    /registry/serviceaccounts/kube-node-lease/default
    /registry/serviceaccounts/kube-public/default
    /registry/serviceaccounts/kube-system/attachdetach-controller
    /registry/serviceaccounts/kube-system/bootstrap-signer
    /registry/serviceaccounts/kube-system/certificate-controller
    /registry/serviceaccounts/kube-system/clusterrole-aggregation-controller
    /registry/serviceaccounts/kube-system/coredns
    /registry/serviceaccounts/kube-system/cronjob-controller
    /registry/serviceaccounts/kube-system/daemon-set-controller
    /registry/serviceaccounts/kube-system/default
    /registry/serviceaccounts/kube-system/deployment-controller
    /registry/serviceaccounts/kube-system/disruption-controller
    /registry/serviceaccounts/kube-system/endpoint-controller
    /registry/serviceaccounts/kube-system/expand-controller
    /registry/serviceaccounts/kube-system/flannel
    /registry/serviceaccounts/kube-system/generic-garbage-collector
    /registry/serviceaccounts/kube-system/horizontal-pod-autoscaler
    /registry/serviceaccounts/kube-system/job-controller
    /registry/serviceaccounts/kube-system/kube-proxy
    /registry/serviceaccounts/kube-system/namespace-controller
    /registry/serviceaccounts/kube-system/node-controller
    /registry/serviceaccounts/kube-system/persistent-volume-binder
    /registry/serviceaccounts/kube-system/pod-garbage-collector
    /registry/serviceaccounts/kube-system/pv-protection-controller
    /registry/serviceaccounts/kube-system/pvc-protection-controller
    /registry/serviceaccounts/kube-system/replicaset-controller
    /registry/serviceaccounts/kube-system/replication-controller
    /registry/serviceaccounts/kube-system/resourcequota-controller
    /registry/serviceaccounts/kube-system/service-account-controller
    /registry/serviceaccounts/kube-system/service-controller
    /registry/serviceaccounts/kube-system/statefulset-controller
    /registry/serviceaccounts/kube-system/token-cleaner
    /registry/serviceaccounts/kube-system/ttl-controller
    /registry/services/endpoints/default/gateway-kong-admin
    /registry/services/endpoints/default/gateway-kong-proxy
    /registry/services/endpoints/default/kubernetes
    /registry/services/endpoints/kube-system/kube-controller-manager
    /registry/services/endpoints/kube-system/kube-dns
    /registry/services/endpoints/kube-system/kube-scheduler
    /registry/services/specs/default/gateway-kong-admin
    /registry/services/specs/default/gateway-kong-proxy
    /registry/services/specs/default/kubernetes
    /registry/services/specs/kube-system/kube-dns

查询某个key的值，--keys-only=false表示要给出value，该参数默认值即为false，所以该参数可以不出现，-w=json表示输出json格式。

    etcdctl get /registry/namespaces/default --prefix --keys-only=false -w=json | python -m json.tool
    {
        "count": 1,
        "header": {
            "cluster_id": 13595082567554063233,
            "member_id": 13154517069409364498,
            "raft_term": 9,
            "revision": 3097040
        },
        "kvs": [
            {
                "create_revision": 147,
                "key": "L3JlZ2lzdHJ5L25hbWVzcGFjZXMvZGVmYXVsdA==",
                "mod_revision": 147,
                "value": "azhzAAoPCgJ2MRIJTmFtZXNwYWNlEl8KRQoHZGVmYXVsdBIAGgAiACokMDY4MWEzM2MtOGM2OC00YjZhLTg5ZmUtZTM3MjI0MzM0NWE3MgA4AEIICLnY0+8FEAB6ABIMCgprdWJlcm5ldGVzGggKBkFjdGl2ZRoAIgA=",
                "version": 1
            }
        ]
    }

以上的key和values都以base64编码了，如果想查看key的值可以执行如下命令。有些value的值包含二进制，不易解开。

    echo "L3JlZ2lzdHJ5L25hbWVzcGFjZXMvZGVmYXVsdA==" | base64 -d
    /registry/namespaces/default


键/registry/apiregistration.k8s.io/apiservices/v1.apiextensions.k8s.io的值不包含二进制，可以直接查询。

    etcdctl get /registry/apiregistration.k8s.io/apiservices/v1.apiextensions.k8s.io --prefix | grep '"kind":"APIService"' | python -m json.tool
    {
        "apiVersion": "apiregistration.k8s.io/v1beta1",
        "kind": "APIService",
        "metadata": {
            "creationTimestamp": "2019-12-14T14:05:42Z",
            "labels": {
                "kube-aggregator.kubernetes.io/automanaged": "onstart"
            },
            "name": "v1.apiextensions.k8s.io",
            "uid": "ac8721f3-81ed-465c-b789-0132ca6e65ae"
        },
        "spec": {
            "group": "apiextensions.k8s.io",
            "groupPriorityMinimum": 16700,
            "service": null,
            "version": "v1",
            "versionPriority": 15
        },
        "status": {
            "conditions": [
                {
                    "lastTransitionTime": "2019-12-14T14:05:42Z",
                    "message": "Local APIServices are always available",
                    "reason": "Local",
                    "status": "True",
                    "type": "Available"
                }
            ]
        }
    }

或者
    
    etcdctl get /registry/apiregistration.k8s.io/apiservices/v1.apiextensions.k8s.io --prefix --keys-only=false -w=json | python -m json.tool
    {
        "count": 1,
        "header": {
            "cluster_id": 13595082567554063233,
            "member_id": 13154517069409364498,
            "raft_term": 9,
            "revision": 3172982
        },
        "kvs": [
            {
                "create_revision": 4,
                "key": "L3JlZ2lzdHJ5L2FwaXJlZ2lzdHJhdGlvbi5rOHMuaW8vYXBpc2VydmljZXMvdjEuYXBpZXh0ZW5zaW9ucy5rOHMuaW8=",
                "mod_revision": 4,
                "value": "eyJraW5kIjoiQVBJU2VydmljZSIsImFwaVZlcnNpb24iOiJhcGlyZWdpc3RyYXRpb24uazhzLmlvL3YxYmV0YTEiLCJtZXRhZGF0YSI6eyJuYW1lIjoidjEuYXBpZXh0ZW5zaW9ucy5rOHMuaW8iLCJ1aWQiOiJhYzg3MjFmMy04MWVkLTQ2NWMtYjc4OS0wMTMyY2E2ZTY1YWUiLCJjcmVhdGlvblRpbWVzdGFtcCI6IjIwMTktMTItMTRUMTQ6MDU6NDJaIiwibGFiZWxzIjp7Imt1YmUtYWdncmVnYXRvci5rdWJlcm5ldGVzLmlvL2F1dG9tYW5hZ2VkIjoib25zdGFydCJ9fSwic3BlYyI6eyJzZXJ2aWNlIjpudWxsLCJncm91cCI6ImFwaWV4dGVuc2lvbnMuazhzLmlvIiwidmVyc2lvbiI6InYxIiwiZ3JvdXBQcmlvcml0eU1pbmltdW0iOjE2NzAwLCJ2ZXJzaW9uUHJpb3JpdHkiOjE1fSwic3RhdHVzIjp7ImNvbmRpdGlvbnMiOlt7InR5cGUiOiJBdmFpbGFibGUiLCJzdGF0dXMiOiJUcnVlIiwibGFzdFRyYW5zaXRpb25UaW1lIjoiMjAxOS0xMi0xNFQxNDowNTo0MloiLCJyZWFzb24iOiJMb2NhbCIsIm1lc3NhZ2UiOiJMb2NhbCBBUElTZXJ2aWNlcyBhcmUgYWx3YXlzIGF2YWlsYWJsZSJ9XX19Cg==",
                "version": 1
            }
        ]
    }

    echo "eyJraW5kIjoiQVBJU2VydmljZSIsImFwaVZlcnNpb24iOiJhcGlyZWdpc3RyYXRpb24uazhzLmlvL3YxYmV0YTEiLCJtZXRhZGF0YSI6eyJuYW1lIjoidjEuYXBpZXh0ZW5zaW9ucy5rOHMuaW8iLCJ1aWQiOiJhYzg3MjFmMy04MWVkLTQ2NWMtYjc4OS0wMTMyY2E2ZTY1YWUiLCJjcmVhdGlvblRpbWVzdGFtcCI6IjIwMTktMTItMTRUMTQ6MDU6NDJaIiwibGFiZWxzIjp7Imt1YmUtYWdncmVnYXRvci5rdWJlcm5ldGVzLmlvL2F1dG9tYW5hZ2VkIjoib25zdGFydCJ9fSwic3BlYyI6eyJzZXJ2aWNlIjpudWxsLCJncm91cCI6ImFwaWV4dGVuc2lvbnMuazhzLmlvIiwidmVyc2lvbiI6InYxIiwiZ3JvdXBQcmlvcml0eU1pbmltdW0iOjE2NzAwLCJ2ZXJzaW9uUHJpb3JpdHkiOjE1fSwic3RhdHVzIjp7ImNvbmRpdGlvbnMiOlt7InR5cGUiOiJBdmFpbGFibGUiLCJzdGF0dXMiOiJUcnVlIiwibGFzdFRyYW5zaXRpb25UaW1lIjoiMjAxOS0xMi0xNFQxNDowNTo0MloiLCJyZWFzb24iOiJMb2NhbCIsIm1lc3NhZ2UiOiJMb2NhbCBBUElTZXJ2aWNlcyBhcmUgYWx3YXlzIGF2YWlsYWJsZSJ9XX19Cg==" | base64 -d | python -m json.tool
    {
        "apiVersion": "apiregistration.k8s.io/v1beta1",
        "kind": "APIService",
        "metadata": {
            "creationTimestamp": "2019-12-14T14:05:42Z",
            "labels": {
                "kube-aggregator.kubernetes.io/automanaged": "onstart"
            },
            "name": "v1.apiextensions.k8s.io",
            "uid": "ac8721f3-81ed-465c-b789-0132ca6e65ae"
        },
        "spec": {
            "group": "apiextensions.k8s.io",
            "groupPriorityMinimum": 16700,
            "service": null,
            "version": "v1",
            "versionPriority": 15
        },
        "status": {
            "conditions": [
                {
                    "lastTransitionTime": "2019-12-14T14:05:42Z",
                    "message": "Local APIServices are always available",
                    "reason": "Local",
                    "status": "True",
                    "type": "Available"
                }
            ]
        }
    }

以下是etcd中存储的kubernetes所有的元数据类型：

    ThirdPartyResourceData
    apiextensions.k8s.io
    apiregistration.k8s.io
    certificatesigningrequests
    clusterrolebindings
    clusterroles
    configmaps
    controllerrevisions
    controllers
    daemonsets
    deployments
    events
    horizontalpodautoscalers
    ingress
    limitranges
    minions
    monitoring.coreos.com
    namespaces
    persistentvolumeclaims
    persistentvolumes
    poddisruptionbudgets
    pods
    ranges
    replicasets
    resourcequotas
    rolebindings
    roles
    secrets
    serviceaccounts
    services
    statefulsets
    storageclasses
    thirdpartyresources