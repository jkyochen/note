## 第一个容器化应用

### 配置文件

+ `nginx-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

#### 解析

**kind** 指定了这个 API 对象的类型（Type），是一个 Deployment。Deployment 是一个定义多副本应用（即多个副本 Pod）的对象。Deployment 还负责在 Pod 定义发生变化时，对每个副本进行滚动更新（Rolling Update）。

**spec.replicas** 副本个数是 2

**spec.template** Pod 模版

**spec.template.spec.containers.image** 容器的镜像是 nginx:1.7.9

**spec.template.spec.containers.ports.containerPort** 容器监听端口是 80

**spec.template.metadata** 每一个 API 对象都有一个叫作 Metadata 的字段，这个字段就是 API 对象的“标识”，即元数据，它也是我们从 Kubernetes 里找到这个对象的主要依据。这其中最主要使用到的字段是 Labels。

**spec.template.metadata.labels** 就是一组 key-value 格式的标签。而像 Deployment 这样的控制器对象，就可以通过这个 Labels 字段从 Kubernetes 中过滤出它所关心的被控制对象。

**spec.selector.matchLabels** 定义标签的过滤规则。我们一般称之为：Label Selector。

**spec.template.metadata.annotations** 专门用来携带 key-value 格式的内部信息。所谓内部信息，指的是对这些信息感兴趣的，是 Kubernetes 组件本身，而不是用户。所以大多数 Annotations，都是在 Kubernetes 运行过程中，被自动加在这个 API 对象上。

一个 Kubernetes 的 API 对象的定义，大多可以分为 Metadata 和 Spec 两个部分。前者存放的是这个对象的元数据，对所有 API 对象来说，这一部分的字段和格式基本上是一样的；而后者存放的，则是属于这个对象独有的定义，用来描述它所要表达的功能。

### 操作

```sh
# run
kubectl create -f nginx-deployment.yaml

# update
kubectl replace -f nginx-deployment.yaml

# run or update
# 声明式 API
kubectl apply -f nginx-deployment.yaml
kubectl get deploy
kubectl describe deploy nginx-deployment

# view
kubectl get pods / kubectl get pods -n default
# 在命令行中，所有 key-value 格式的参数，都使用“=”而非“:”表示。
kubectl get pods -l app=nginx
kubectl describe pod nginx-deployment-54f57cf6bf-pljd4

# join pod
kubectl exec -it nginx-deployment-54f57cf6bf-pljd4 -- /bin/bash

# delete
kubectl delete -f nginx-deployment.yaml
```

在 kubectl describe 命令返回的结果中，你可以清楚地看到这个 Pod 的详细信息，比如它的 IP 地址等等。其中，有一个部分值得你特别关注，它就是Events（事件）。
这个部分正是我们将来进行 Debug 的重要依据。如果有异常发生，你一定要第一时间查看这些 Events，往往可以看到非常详细的错误信息。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.8
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: nginx-vol
      volumes:
      - name: nginx-vol
        emptyDir: {}
```

在 Deployment 的 Pod 模板部分添加了一个 volumes 字段，定义了这个 Pod 声明的所有 Volume。它的名字叫作 nginx-vol，类型是 emptyDir。

emptyDir 就等同于我们之前讲过的 Docker 的隐式 Volume 参数，即：不显式声明宿主机目录的 Volume。Kubernetes 也会在宿主机上创建一个临时目录，这个目录将来就会被绑定挂载到容器所声明的 Volume 目录上。
Kubernetes 的 emptyDir 类型，只是把 Kubernetes 创建的临时目录作为 Volume 的宿主机目录，交给了 Docker。这么做的原因，是 Kubernetes 不想依赖 Docker 自己创建的那个 `_data` 目录。

而 Pod 中的容器，使用的是 volumeMounts 字段来声明自己要挂载哪个 Volume，并通过 mountPath 字段来定义容器内的 Volume 目录，比如：/usr/share/nginx/html。


Kubernetes 也提供了显式的 Volume 定义，它叫做 hostPath。

```yaml
...
   volumes:
     - name: nginx-vol
       hostPath:
         path: /var/data
```

### pod

Pod 就是 Kubernetes 世界里的“应用”；而一个应用，可以由多个容器组成。

这样使用一种 API 对象（Deployment）管理另一种 API 对象（Pod）的方法，在 Kubernetes 中，叫作“控制器”模式（controller pattern）。

Kubernetes 推荐的使用方式，是用一个 YAML 文件来描述你所要部署的 API 对象。然后，统一使用 kubectl apply 命令完成对这个对象的创建和更新操作。

在 Kubernetes 中，我们经常会看到它通过一种 API 对象来管理另一种 API 对象，比如 Deployment 和 Pod 之间的关系；而由于 Pod 是“最小”的对象，所以它往往都是被其他对象控制的。这种组合方式，正是 Kubernetes 进行容器编排的重要模式。

