# etcd 3.3.11安装和使用介绍

- ### 环境准备

Linux版本

    uname -a
    Linux etcd1 3.10.0-1062.9.1.el7.x86_64 #1 SMP Fri Dec 6 15:49:49 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
    
三个节点

    cat /etc/hosts
    192.168.1.66 etcd1
    192.168.1.67 etcd2
    192.168.1.68 etcd3

关闭防火墙

    systemctl stop firewalld
    systemctl disable firewalld

- ### 安装

安装etcd。

    yum install -y etcd

配置etcd1：

    #[Member]
    #ETCD_CORS=""
    ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
    #ETCD_WAL_DIR=""
    ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
    ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,http://0.0.0.0:4001"
    #ETCD_MAX_SNAPSHOTS="5"
    #ETCD_MAX_WALS="5"
    ETCD_NAME="etcd1"
    #ETCD_SNAPSHOT_COUNT="100000"
    #ETCD_HEARTBEAT_INTERVAL="100"
    #ETCD_ELECTION_TIMEOUT="1000"
    #ETCD_QUOTA_BACKEND_BYTES="0"
    #ETCD_MAX_REQUEST_BYTES="1572864"
    #ETCD_GRPC_KEEPALIVE_MIN_TIME="5s"
    #ETCD_GRPC_KEEPALIVE_INTERVAL="2h0m0s"
    #ETCD_GRPC_KEEPALIVE_TIMEOUT="20s"
    #
    #[Clustering]
    ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.1.66:2380"
    ETCD_ADVERTISE_CLIENT_URLS="http://192.168.1.66:2379,http://192.168.1.66:4001"
    #ETCD_DISCOVERY=""
    #ETCD_DISCOVERY_FALLBACK="proxy"
    #ETCD_DISCOVERY_PROXY=""
    #ETCD_DISCOVERY_SRV=""
    ETCD_INITIAL_CLUSTER="etcd1=http://192.168.1.66:2380,etcd2=http://192.168.1.67:2380,etcd3=http://192.168.1.68:2380"
    ETCD_INITIAL_CLUSTER_TOKEN="etcd-twingao-cluster"
    ETCD_INITIAL_CLUSTER_STATE="new"
    #ETCD_STRICT_RECONFIG_CHECK="true"
    #ETCD_ENABLE_V2="true"

配置etcd2：

    #[Member]
    #ETCD_CORS=""
    ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
    #ETCD_WAL_DIR=""
    ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
    ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,http://0.0.0.0:4001"
    #ETCD_MAX_SNAPSHOTS="5"
    #ETCD_MAX_WALS="5"
    ETCD_NAME="etcd2"
    #ETCD_SNAPSHOT_COUNT="100000"
    #ETCD_HEARTBEAT_INTERVAL="100"
    #ETCD_ELECTION_TIMEOUT="1000"
    #ETCD_QUOTA_BACKEND_BYTES="0"
    #ETCD_MAX_REQUEST_BYTES="1572864"
    #ETCD_GRPC_KEEPALIVE_MIN_TIME="5s"
    #ETCD_GRPC_KEEPALIVE_INTERVAL="2h0m0s"
    #ETCD_GRPC_KEEPALIVE_TIMEOUT="20s"
    #
    #[Clustering]
    ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.1.67:2380"
    ETCD_ADVERTISE_CLIENT_URLS="http://192.168.1.67:2379,http://192.168.1.67:4001"
    #ETCD_DISCOVERY=""
    #ETCD_DISCOVERY_FALLBACK="proxy"
    #ETCD_DISCOVERY_PROXY=""
    #ETCD_DISCOVERY_SRV=""
    ETCD_INITIAL_CLUSTER="etcd1=http://192.168.1.66:2380,etcd2=http://192.168.1.67:2380,etcd3=http://192.168.1.68:2380"
    ETCD_INITIAL_CLUSTER_TOKEN="etcd-twingao-cluster"
    ETCD_INITIAL_CLUSTER_STATE="new"
    #ETCD_STRICT_RECONFIG_CHECK="true"
    #ETCD_ENABLE_V2="true"

配置etcd3：

    #[Member]
    #ETCD_CORS=""
    ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
    #ETCD_WAL_DIR=""
    ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
    ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,http://0.0.0.0:4001"
    #ETCD_MAX_SNAPSHOTS="5"
    #ETCD_MAX_WALS="5"
    ETCD_NAME="etcd3"
    #ETCD_SNAPSHOT_COUNT="100000"
    #ETCD_HEARTBEAT_INTERVAL="100"
    #ETCD_ELECTION_TIMEOUT="1000"
    #ETCD_QUOTA_BACKEND_BYTES="0"
    #ETCD_MAX_REQUEST_BYTES="1572864"
    #ETCD_GRPC_KEEPALIVE_MIN_TIME="5s"
    #ETCD_GRPC_KEEPALIVE_INTERVAL="2h0m0s"
    #ETCD_GRPC_KEEPALIVE_TIMEOUT="20s"
    #
    #[Clustering]
    ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.1.68:2380"
    ETCD_ADVERTISE_CLIENT_URLS="http://192.168.1.68:2379,http://192.168.1.68:4001"
    #ETCD_DISCOVERY=""
    #ETCD_DISCOVERY_FALLBACK="proxy"
    #ETCD_DISCOVERY_PROXY=""
    #ETCD_DISCOVERY_SRV=""
    ETCD_INITIAL_CLUSTER="etcd1=http://192.168.1.66:2380,etcd2=http://192.168.1.67:2380,etcd3=http://192.168.1.68:2380"
    ETCD_INITIAL_CLUSTER_TOKEN="etcd-twingao-cluster"
    ETCD_INITIAL_CLUSTER_STATE="new"
    #ETCD_STRICT_RECONFIG_CHECK="true"
    #ETCD_ENABLE_V2="true"

参数说明：

    ETCD_DATA_DIR：数据存储目录
    ETCD_LISTEN_PEER_URLS：与其他节点通信时的监听地址列表，通信协议可以是http、https
    ETCD_LISTEN_CLIENT_URLS：与客户端通信时的监听地址列表
    ETCD_NAME：节点名称
    ETCD_INITIAL_ADVERTISE_PEER_URLS：节点在整个集群中的通信地址列表，可以理解为能与外部通信的ip端口
    ETCD_ADVERTISE_CLIENT_URLS：告知集群中其他成员自己名下的客户端的地址列表
    ETCD_INITIAL_CLUSTER：集群内所有成员的地址，这就是为什么称之为静态发现，因为所有成员的地址都必须配置
    ETCD_INITIAL_CLUSTER_TOKEN：初始化集群口令，用于标识不同集群
    ETCD_INITIAL_CLUSTER_STATE：初始化集群状态，new表示新建

启动etcd服务，并配置自启动。

    systemctl start etcd
    systemctl enable etcd

> 如果不能启动etcd服务，可以删除所有节点/var/lib/etcd目录下内容，然后重新启动。

查看etcd版本。

    etcd --version
    etcd Version: 3.3.11
    Git SHA: 2cf9e51
    Go Version: go1.10.3
    Go OS/Arch: linux/amd64

查看etcdctl版本。

    etcdctl --version
    etcdctl version: 3.3.11
    API version: 2

etcd api分为2和3版本，其对应的命令行客户端etcdctl也有两种版本模式，两个版本在使用上有些区别，从上面的命令看安装后etcd api缺省是2版本，采用如下方法可以改为3版本。

    vi /etc/profile
    export ETCDCTL_API=3
    source /etc/profile

- ### 使用

查看集群节点：

    etcdctl member list
    3bfc158cf02dfab9, started, etcd3, http://192.168.1.68:2380, http://192.168.1.68:2379,http://192.168.1.68:4001
    8835dd4fc48bb08a, started, etcd2, http://192.168.1.67:2380, http://192.168.1.67:2379,http://192.168.1.67:4001
    fbe83a7120c1fef4, started, etcd1, http://192.168.1.66:2380, http://192.168.1.66:2379,http://192.168.1.66:4001

在etcd1节点停止etcd服务，再查看集群，发现无法连接，因为etcdctl缺省访问本地的etcd服务。

    systemctl stop etcd
    etcdctl member list
    Error: dial tcp 127.0.0.1:2379: connect: connection refused

在命令行指定endpoint，etcdctl可以向其他endpoint请求数据，防止一个节点出故障导致集群无法访问。

    etcdctl --endpoints=192.168.1.66:2379,192.168.1.67:2379,192.168.1.68:2379 member list
    3bfc158cf02dfab9, started, etcd3, http://192.168.1.68:2380, http://192.168.1.68:2379,http://192.168.1.68:4001
    8835dd4fc48bb08a, started, etcd2, http://192.168.1.67:2380, http://192.168.1.67:2379,http://192.168.1.67:4001
    fbe83a7120c1fef4, started, etcd1, http://192.168.1.66:2380, http://192.168.1.66:2379,http://192.168.1.66:4001

查看endpoint的状态。

    etcdctl --endpoints=192.168.1.66:2379,192.168.1.67:2379,192.168.1.68:2379 endpoint status
    Failed to get the status of endpoint 192.168.1.66:2379 (context deadline exceeded)
    192.168.1.67:2379, 8835dd4fc48bb08a, 3.3.11, 20 kB, false, 16, 20
    192.168.1.68:2379, 3bfc158cf02dfab9, 3.3.11, 20 kB, true, 16, 20

endpoint健康检查。

    etcdctl --endpoints=192.168.1.66:2379,192.168.1.67:2379,192.168.1.68:2379 endpoint health
    192.168.1.68:2379 is healthy: successfully committed proposal: took = 6.832804ms
    192.168.1.67:2379 is healthy: successfully committed proposal: took = 10.387798ms
    192.168.1.66:2379 is unhealthy: failed to connect: dial tcp 192.168.1.66:2379: connect: connection refused
    Error: unhealthy cluster

etcdctl支持的3版本的命令。

    etcdctl
    NAME:
            etcdctl - A simple command line client for etcd3.
    
    USAGE:
            etcdctl
    
    VERSION:
            3.3.11
    
    API VERSION:
            3.3
    
    
    COMMANDS:
            get                     Gets the key or a range of keys
            put                     Puts the given key into the store
            del                     Removes the specified key or range of keys [key, range_end)
            txn                     Txn processes all the requests in one transaction
            compaction              Compacts the event history in etcd
            alarm disarm            Disarms all alarms
            alarm list              Lists all alarms
            defrag                  Defragments the storage of the etcd members with given endpoints
            endpoint health         Checks the healthiness of endpoints specified in `--endpoints` flag
            endpoint status         Prints out the status of endpoints specified in `--endpoints` flag
            endpoint hashkv         Prints the KV history hash for each endpoint in --endpoints
            move-leader             Transfers leadership to another etcd cluster member.
            watch                   Watches events stream on keys or prefixes
            version                 Prints the version of etcdctl
            lease grant             Creates leases
            lease revoke            Revokes leases
            lease timetolive        Get lease information
            lease list              List all active leases
            lease keep-alive        Keeps leases alive (renew)
            member add              Adds a member into the cluster
            member remove           Removes a member from the cluster
            member update           Updates a member in the cluster
            member list             Lists all members in the cluster
            snapshot save           Stores an etcd node backend snapshot to a given file
            snapshot restore        Restores an etcd member snapshot to an etcd directory
            snapshot status         Gets backend snapshot status of a given file
            make-mirror             Makes a mirror at the destination etcd cluster
            migrate                 Migrates keys in a v2 store to a mvcc store
            lock                    Acquires a named lock
            elect                   Observes and participates in leader election
            auth enable             Enables authentication
            auth disable            Disables authentication
            user add                Adds a new user
            user delete             Deletes a user
            user get                Gets detailed information of a user
            user list               Lists all users
            user passwd             Changes password of user
            user grant-role         Grants a role to a user
            user revoke-role        Revokes a role from a user
            role add                Adds a new role
            role delete             Deletes a role
            role get                Gets detailed information of a role
            role list               Lists all roles
            role grant-permission   Grants a key to a role
            role revoke-permission  Revokes a key from a role
            check perf              Check the performance of the etcd cluster
            help                    Help about any command
    
    OPTIONS:
          --cacert=""                               verify certificates of TLS-enabled secure servers using this CA bundle
          --cert=""                                 identify secure client using this TLS certificate file
          --command-timeout=5s                      timeout for short running command (excluding dial timeout)
          --debug[=false]                           enable client-side debug logging
          --dial-timeout=2s                         dial timeout for client connections
      -d, --discovery-srv=""                        domain name to query for SRV records describing cluster endpoints
          --endpoints=[127.0.0.1:2379]              gRPC endpoints
      -h, --help[=false]                            help for etcdctl
          --hex[=false]                             print byte strings as hex encoded strings
          --insecure-discovery[=true]               accept insecure SRV records describing cluster endpoints
          --insecure-skip-tls-verify[=false]        skip server certificate verification
          --insecure-transport[=true]               disable transport security for client connections
          --keepalive-time=2s                       keepalive time for client connections
          --keepalive-timeout=6s                    keepalive timeout for client connections
          --key=""                                  identify secure client using this TLS key file
          --user=""                                 username[:password] for authentication (prompt if password is not supplied)
      -w, --write-out="simple"                      set the output format (fields, json, protobuf, simple, table)

创建key

    etcdctl put hello world
    OK

查询给定key的值。第一行为key，第二行为value。

    etcdctl get hello
    hello
    world

在etcd1节点监听key hello值的变化，该命令会一直等待。

    etcdctl watch hello

在etcd2节点修改该key的值。

    etcdctl put hello world!
    OK

在etcd1节点监听命令立马就有反馈。

    etcdctl watch hello
    PUT
    hello
    world!

先增加几个key。

    etcdctl put hello1 nihao
    OK
    etcdctl put hello2 ninhao
    OK

基于key前缀查找。

    etcdctl get hello --prefix
    hello
    world!
    hello1
    nihao
    hello2
    ninhao

查找所有的key值。

    etcdctl put key1 value1
    OK
    etcdctl get "" --prefix
    hello
    world!
    hello1
    nihao
    hello2
    ninhao
    key1
    value1

