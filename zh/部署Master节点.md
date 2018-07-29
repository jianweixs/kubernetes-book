## 部署Master节点

kubernetes master 节点包含的组件：

- kube-apiserver
- kube-scheduler
- kube-controller-manager
  目前这三个组件需要部署在同一台机器上。
  kube-scheduler、kube-controller-manager 和 kube-apiserver 三者的功能紧密相关；
  同时只能有一个 kube-scheduler、kube-controller-manager 进程处于工作状态，如果运行多个，则需要通过选举产生一个 leader；

server 的 tarball kubernetes-server-linux-amd64.tar.gz 已经包含了 client(kubectl) 二进制文件，所以不用单独下载kubernetes-client-linux-amd64.tar.gz文件；

```
# wget https://dl.k8s.io/v1.6.0/kubernetes-client-linux-amd64.tar.gz
# wget https://dl.k8s.io/v1.6.0/kubernetes-server-linux-amd64.tar.gz
# tar -xzvf kubernetes-server-linux-amd64.tar.gz
# cd kubernetes
# tar -xzvf  kubernetes-src.tar.gz
```

将二进制文件拷贝到指定路径

```
# cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/bin/
```

配置和启动 kube-apiserver
创建 kube-apiserver的service配置文件
service配置文件/usr/lib/systemd/system/kube-apiserver.service内容：

```
[Unit]
Description=Kubernetes API Service
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/apiserver
ExecStart=/usr/local/bin/kube-apiserver \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_ETCD_SERVERS \
        $KUBE_API_ADDRESS \
        $KUBE_API_PORT \
        $KUBELET_PORT \
        $KUBE_ALLOW_PRIV \
        $KUBE_SERVICE_ADDRESSES \
        $KUBE_ADMISSION_CONTROL \
        $KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

/etc/kubernetes/config文件的内容为：

```
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"

# How the controller-manager, scheduler, and proxy find the apiserver
#KUBE_MASTER="--master=http://test-001.jimmysong.io:8080"
KUBE_MASTER="--master=http://172.20.0.113:8080"
```

该配置文件同时被kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy使用。
apiserver配置文件/etc/kubernetes/apiserver内容为：

```
###
## kubernetes system config
##
## The following values are used to configure the kube-apiserver
##
#
## The address on the local server to listen to.
KUBE_API_ADDRESS="--advertise-address=10.71.10.188 --bind-address=0.0.0.0 --insecure-bind-address=0.0.0.0"
#
## The port on the local server to listen on.
KUBE_API_PORT="--insecure-port=8080 --secure-port=6443"
#
## Port minions listen on
#KUBELET_PORT="--kubelet-port=10250"
#
## Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=http://10.71.10.188:2379"
#
## Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
#
## default admission control policies
KUBE_ADMISSION_CONTROL="--admission-control=ServiceAccount,NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota,NodeRestriction,DefaultStorageClass"
#
## Add your own!

KUBE_API_ARGS="--authorization-mode=RBAC,Node \
                --cert-dir=/etc/kubernetes/ssl \
                --log-dir=/data1/kubernetes/logs \
                --kubelet-https=true \
                --enable-bootstrap-token-auth \
                --token-auth-file=/etc/kubernetes/token.csv \
                --service-node-port-range=30000-32767 \
                --client-ca-file=/etc/kubernetes/ssl/ca.pem \
                --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
                --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
                --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
                --enable-swagger-ui=true \
                --apiserver-count=3 \
                --audit-log-maxage=30 \
                --audit-log-maxbackup=3 \
                --audit-log-maxsize=100 \
                --audit-log-path=/var/lib/audit.log \
                --event-ttl=1h "
```

- --experimental-bootstrap-token-auth Bootstrap Token Authentication在1.9版本已经变成了正式feature，参数名称改为--enable-bootstrap-token-auth
  如果中途修改过--service-cluster-ip-range地址，则必须将default命名空间的kubernetes的service给删除，使用命令：kubectl delete service kubernetes，然后系统会自动用新的ip重建这个service，不然apiserver的log有报错the cluster IP x.x.x.x for service kubernetes/default is not within the service CIDR x.x.x.x/16; please recreate
- --authorization-mode=RBAC 指定在安全端口使用 RBAC 授权模式，拒绝未通过授权的请求；
  kube-scheduler、kube-controller-manager 一般和 kube-apiserver 部署在同一台机器上，它们使用非安全端口和 kube-apiserver通信;
  kubelet、kube-proxy、kubectl 部署在其它 Node 节点上，如果通过安全端口访问 kube-apiserver，则必须先通过 TLS 证书认证，再通过 RBAC 授权；kube-proxy、kubectl 通过在使用的证书里指定相关的 User、Group 来达到通过 RBAC 授权的目的；
  如果使用了 kubelet TLS Boostrap 机制，则不能再指定 --kubelet-certificate-authority、* --kubelet-client-certificate 和 --kubelet-client-key 选项，否则后续 kube-apiserver 校验 kubelet 证书时出现 ”x509: certificate signed by unknown authority“ 错误；
- --admission-control 值必须包含 ServiceAccount；
- --bind-address 不能为 127.0.0.1；
- --runtime-config配置为rbac.authorization.k8s.io/v1beta1，表示运行时的apiVersion；
- --service-cluster-ip-range 指定 Service Cluster IP 地址段，该地址段不能路由可达；
  缺省情况下 kubernetes 对象保存在 etcd /registry 路径下，可以通过 --etcd-prefix 参数进行调整；
  如果需要开通http的无认证的接口，则可以增加以下两个参数：--insecure-port=8080 --insecure-bind-address=127.0.0.1。注意，生产上不要绑定到非127.0.0.1的地址上