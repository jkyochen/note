## [Kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)

要真正发挥容器技术的实力，你就不能仅仅局限于对 Linux 容器本身的钻研和使用。

```sh
# 创建一个 Master 节点
kubeadm init

# 将一个 Node 节点加入到当前集群中
kubeadm join <Master 节点的 IP 和端口 >
```

### kubeadm 的工作原理

把 kubelet 直接运行在宿主机上，然后使用容器部署其他的 Kubernetes 组件。

### kubeadm init 的工作流程

1. 首先是一系列的检查工作，以确定这台机器可以用来部署 Kubernetes。
    + [Preflight Checks](https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/#preflight-checks)

2. 生成 Kubernetes 对外提供服务所需的各种证书和对应的目录。
    + Kubernetes 对外提供服务时，除非专门开启“不安全模式”，否则都要通过 HTTPS 才能访问 kube-apiserver。这就需要为 Kubernetes 集群配置好证书文件。
    + kubeadm 为 Kubernetes 项目生成的证书文件都放在 Master 节点的 /etc/kubernetes/pki 目录下。在这个目录下，最主要的证书文件是 ca.crt 和对应的私钥 ca.key。
    + 用户使用 kubectl 获取容器日志等 streaming 操作时，需要通过 kube-apiserver 向 kubelet 发起请求，这个连接也必须是安全的。kubeadm 为这一步生成的是 apiserver-kubelet-client.crt 文件，对应的私钥是 apiserver-kubelet-client.key。
    + 可以选择不让 kubeadm 为你生成这些证书，而是拷贝现有的证书到如下证书的目录里：`/etc/kubernetes/pki/ca.{crt,key}`

3. kubeadm 接下来会为其他组件生成访问 kube-apiserver 所需的配置文件。

    ```sh
    # 记录的是当前这个 Master 节点的服务器地址、监听端口、证书目录等信息。
    ls /etc/kubernetes/
    admin.conf  controller-manager.conf  kubelet.conf  scheduler.conf
    ```

4. kubeadm 会为 Master 组件生成 Pod 配置文件。

     + Kubernetes 有三个 Master 组件 kube-apiserver、kube-controller-manager、kube-scheduler，而它们都会被使用 Pod 的方式部署起来。
    + 在 Kubernetes 中，有一种特殊的容器启动方法叫做“Static Pod”。它允许你把要部署的 Pod 的 YAML 文件放在一个指定的目录里。这样，当这台机器上的 kubelet 启动时，它会自动检查这个目录，加载所有的 Pod YAML 文件，然后在这台机器上启动它们。
    + 在 kubeadm 中，Master 组件的 YAML 文件会被生成在 /etc/kubernetes/manifests 路径下。
    + kube-apiserver.yaml
        1. 这个 Pod 里只定义了一个容器，它使用的镜像是：k8s.gcr.io/kube-apiserver-amd64:v1.11.1 。这个镜像是 Kubernetes 官方维护的一个组件镜像。
        2. 这个容器的启动命令（commands）是 kube-apiserver --authorization-mode=Node,RBAC …，这样一句非常长的命令。其实，它就是容器里 kube-apiserver 这个二进制文件再加上指定的配置参数而已。
        3. 如果你要修改一个已有集群的 kube-apiserver 的配置，需要修改这个 YAML 文件。
        4. 这些组件的参数也可以在部署时指定。

5. kubeadm 还会再生成一个 Etcd 的 Pod YAML 文件，用来通过同样的 Static Pod 的方式启动 Etcd。`/etc/kubernetes/manifests/etcd.yaml`

6. Master 容器启动后，kubeadm 会通过检查 `localhost:6443/healthz` 这个 Master 组件的健康检查 URL，等待 Master 组件完全运行起来。

7. 然后，kubeadm 就会为集群生成一个 bootstrap token。

    + 在后面，只要持有这个 token，任何一个安装了 kubelet 和 kubadm 的节点，都可以通过 kubeadm join 加入到这个集群当中。
    + 在 token 生成之后，kubeadm 会将 ca.crt 等 Master 节点的重要信息，通过 ConfigMap 的方式保存在 Etcd 当中，供后续部署 Node 节点使用。这个 ConfigMap 的名字是 cluster-info。

7. 最后一步，就是安装默认插件。
    + Kubernetes 默认 kube-proxy 和 DNS 这两个插件是必须安装的。它们分别用来提供整个集群的服务发现和 DNS 功能。

## kubeadm join 的工作流程

因为，任何一台机器想要成为 Kubernetes 集群中的一个节点，就必须在集群的 kube-apiserver 上注册。可是，要想跟 apiserver 打交道，这台机器就必须要获取到相应的证书文件（CA 文件）。可是，为了能够一键安装，我们就不能让用户去 Master 节点上手动拷贝这些文件。

所以，kubeadm 至少需要发起一次“不安全模式”的访问到 kube-apiserver，从而拿到保存在 ConfigMap 中的 cluster-info（它保存了 APIServer 的授权信息）。而 bootstrap token，扮演的就是这个过程中的安全验证的角色。

只要有了 cluster-info 里的 kube-apiserver 的地址、端口、证书，kubelet 就可以以“安全模式”连接到 apiserver 上，这样一个新的节点就部署完成了。

## 配置 kubeadm 的部署参数

```
kubeadm init --config kubeadm.yaml

# default init configuration
# kubeadm config print init-defaults
```
