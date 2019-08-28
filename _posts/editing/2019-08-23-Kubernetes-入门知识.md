# Kubernetes 
## Cheatsheet
```
kubectl get | edit | delete | create | logs | set | apply
	ns/namespaces
	po/pods
	no/nodes
	cs/componentstatuses
	deployments
	ev/events
	ep/endpoints
	ing/ingresses
	jobs
	quota/resourcequotas
	rs/replicationcontrollers
	secrets
	svc/services
# run a commond based in the container based on image-name
kubectl exec -it container-id bash
# 获得发布 service/deployment 等的 yaml
kubectl get svc/svc-id -o yaml

# 批量删除状态异常的 pod
kubectl get po -n ns | grep Status | awk {'print $1'} | xargs -I {} kubectl delete po {} -n ns
# 强制删除
kubectl delete pods pod-name --force --grace-period=0 -n ns
```
## K8S 集群
K8S 集群实际上就是在许多物理机或虚拟机构成的节点上部署 K8S 进程，协调工作。Master 节点安装部署 etcd, kube-apiserver, kube-controller, kube-scheduler 等服务进程。用户使用 kubectl 作为客户端与 Master 节点进行交互。在工作节点上只需部署 kubelet 和 kube-proxy 服务进程。这些机器与服务进程一起实现容器编排，自动化应用部署与升级等功能。
## 深入理解 Pod
Pod 运行在 Node 上，通常一个节点上运行许多 Pod。容器（例如 Docker, CoreOS 等，本文以 Docker 为其容器）运行在 Pod 里。每个 Pod 里都运行着一个特殊的、业务无关的 Pause 容器(Container，Pause 容器代表整个容器组的状态)，其它的容器则为业务容器，业务容器共享 Pause 容器的网络栈和 Volume 挂载卷，因此同一个 Pod 里的容器之间通信和数据交换十分高效。但是实际上，每个 Pod 里往往只运行一个业务容器，除非某些业务具有密切关联，我们才将它们放在同一个 Pod 里。Pod 和 Container 的关系，Pod 相当于虚拟机，Container 相当于虚拟机里的进程，**同一个 Pod 中**所有 Container 的 IP 地址都是 Pod 的 IP 地址，Container 共享 Pod 的端口空间，类似进程通信，Container 之间通过 localhost:port 就能相互访问。Pod IP:container-port (实际占用 Pod 的 port) 构成了 Endpoint 的概念。Pod IP 是由集群统一分配的（**etcd** [How Does Kubernetes Use etcd](https://matthewpalmer.net/kubernetes-app-developer/articles/how-does-kubernetes-use-etcd.html)），不同 Pod 内的容器可以直接使用 Pod IP 相互通信。简单来说，etcd 是一个分布一致性的键值数据存储系统（较于 Zookeeper 相对简单，常用于共享配置和服务发现），k8s 用 etcd 来存储元数据。
可以基于 yaml 文件创建 Pod，示例如下。
```
apiVersion: v1
kind: Pod
metadata:
  name: myweb-service
  namespace: default
  labels: 
    name: myweb-service
spec:
  containers:
  - name: myweb-service
    image: docker-registry-1.com/myweb-service-image-name:version
    imagePullPolicy: Always
    ports:
      - containerPort: 8080
    env:
      - name: REDIS_SERVICE_HOST
        value: 'redis'
  - name: myweb-service-coworker
    image: docker-registry-2.com/myweb-service-coworker-image-name:version
    ports:
      - containerPort: 3306
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  imagePullSecrets:
    - name: registry-1-secret    
    - name: registry-2-secret
```
使用 `kubectl create -f pod.yaml` 命令即可创建相应的 Pod。创建过程大致为：kubectl 客户端通过 Socket 连接向 Master 节点 kube-apiserver 进程发送创建 Pod 请求，kube-apiserver 进程接到请求后将 pod 信息存储到 etcd。接下来由 Master 节点的 kube-controller-mangager 进程负责实现 Pod 相关的副本策略、Node 选择、配额管理、Namespace、ServiceAccount 管理、Token、Service 适配、Endpoint 注册等。再接下来就由 Master 节点的 kube-scheduler 进程负责为 Pod 的创建任务选择最合适目标 Node，并将之后的工作交由 Node 节点上的 kubelet 进程实施。Node 节点上的 kubelet 进程会将 Pod 实例化为一组相关的 Docker 容器。
具体到上面的 yaml 文件，
1) K8S 集群最终会选择一个 Node 启动该 Pod，Pod name 是使用 `kubectl get pod` 时能看到的 pod 名
2) label 则随后可以通过 label selector 进行选择，例如在下文的 Service yaml 文件中，使用 `selector name: myweb-service` 即可选定这里的 pod。一个 label 是一个键值对，key 和 value 都可以根据使用场景自由定义
3) 这个 Pod 里预期要运行三个容器，一个默认的 pause 容器，两个自定义的容器 myweb-service 和 myweb-service-coworker，两个自定义容器分别使用 8080 和 3306 提供服务
4) 启动自定义容器时，Node 节点的 docker 进程将从 docker-registry-1.com 和 docker-registry-2.com 分别拉取镜像，拉取镜像时，如果需要认证则分别从 registry-1-secret 和 registry-2-secret 获取。需要注意的是 registry-1-secret 和 registry-2-secret 并非简单的用户名/密码信息，而是我们提前在 K8S 集群创建好的一种 Secret 类型，它还包含了该 Secret 对应的 registry 是哪个等信息。创建相应 Secret 的命令例如 `kubectl create secret docker-registry <name> --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL`
5) 如果 Pod 的 yaml 文件里没有 imagePullSecrets 并且相应的镜像 registry 需要认证信息，那么 Node 节点创建 Pod 时会试图使用 Pod 的 ServiceAccount 中配置的 imagePullSecret，如果没有找到或者认证失败，则无法创建 Pod。我们之前提到过，Pod 类似是 Container 的虚拟机，Container 类似是 Pod 虚拟机中的进程，那么 ServiceAccount 就类似进程在虚拟机运行时的身份信息（用户）。创建 Pod 时如果不指定 ServiceAccount，则使用默认的 ServiceAccount (default)。
6) 可以限制 container 在 Pod 中运行时可使用的 CPU 和 Memory 量。

上面 Pod 的 yaml 文件只表明 Pod 相关的创建信息，但基本上我们不会直接创建 Pod，也不会直接使用 Pod 提供服务。实际应用中，我们常通过创建 Depolyment 来间接创建 Pod，通过 Service 来适配 Pod 提供服务。此外，也可以使用 helm 来部署项目到 Kubernetes 集群中，helm 是 Kubernetes 的包管理器，当我们需要将一个大型项目发布到 K8S 时，往往涉及到许多镜像/容器、服务等的发布，这时可以借助 helm 工具将所有的相关项目组合到一起进行发布，简化发布流程，并方便以后的更新维护。

## Deployment
Deployment 是对旧版本 ReplicationController 的升级，对 ReplicaSet 的封装。这里需要注意 DaemonSet 和 ReplicaSet 的区别。
DaemonSet 要求每个相关节点上都运行一个关联 Pod 的副本，有新的 Node 加入集群时，DaemonSet 关联 Pod 会在新 Node 上创建，例如 **kube-proxy**(存疑？？？) 进程就可以以 DaemonSet 形式每个节点上运行的。ReplicaSet 要保证指定数量的 Pod 运行在集群中，并不关心其运行在哪些节点（一个节点上可能运行多个 ReplicaSet 的关联 Pod）。
一个 Deployment 的 yaml 文件如下。
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels: 
    name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
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
1) 使用 `kubectl create -f deployment.yaml` 创建 Deployment 后，可以使用 `kubectl get rs` 看到集群中创建了对应的 ReplicaSet，副本数量由 .spec.relicas 控制
2) .spec.template 的内容实际上就是 pod yaml 的内容，只是少了 apiVersion 和 kind 项
3) .spec.selector 定义了这个 deployment 是管理哪些 pod 的，因此其筛选规则要匹配 .spec.template 中 pod 的 label 定义
## 深入理解 Service
虽然不同 Pod 内的容器可以直接使用 Pod IP 相互通信，但是，我们的业务容器需要访问另一个业务容器的服务时，通常不会关心目标业务容器被分配到了哪一个 Pod 中；在目标业务的运行过程中，当 Pod 由于某种原因被销毁时，目标业务容器也可能会漂移到另外的 Pod 上；同时，HA 需求会导致目标业务容器可能会在多个 Pod 上同时提供服务，如果直接使用 Pod IP 访问业务容器，还可能要求我们考虑负载均衡策略。基于以上原因，一个业务需要访问另一个业务的服务时，应通过 Service 来访问。
Service 是 Kubernetes 的核心概念，通过创建 Service，可以为一组具有相同功能的容器应用提供一个统一的入口地址，并且将请求进行负载均衡后分发到 Service 关联的每个 Endpoint 上。Service 使用 Label Selector 关联到其后端 Pod 上。 

Service 可以看作是 Pod 提供服务的接口层。K8S 集群内部 Pod 里的应用访问 Service 和集群外部应用访问 Service 有较大的不同。

可以使用 kubectl expose 命令创建 Service；也可以先通过配置文件定义 Service，再基于 yaml 文件通过 kubectl create 命令创建 Service。
```
apiVersion: v1
kind: Service
metadata:
  labels:
    name: my-app
  name: my-app
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30062
  selector:
    name: my-app
```
- spec.type: NodePort
上面的 yaml 文件中 spec.type 指定集群对外（集群外部，可能是外网，也可能是内网）暴露该服务，nodePort 指定为 30062（也可以不指定，有集群自动分配）。该 Service 被创建后，集群中每个 node 都会开放 nodePort 30062。每个 k8s 集群 worker 节点都是运行着 kube-proxy 进程，该进程监听某个特定端口的网络流量，当使用任意一个 <nodeIP>:nodePort 进行访问时，请求会被路由到同一节点 kube-proxy 监听的端口，kube-proxy 进程再将流量导向到 service endpoint 的某一个 pod 上。在集群内部，仍然可以使用 Service 的 <clusterIP>:port 访问服务。
- spec.type: LoadBalancer
```
spec:
  type: LoadBalancer
  ports:
  - port: 8080
    targetPort: 8080
```
以上述方式创建的 Service，查看时会有如下内容
```
$ kubectl get svc influxdb
NAME      CLUSTER-IP     EXTERNAL-IP     PORT(S)         AGE
my-app   10.97.121.42   10.13.242.236   8080:30051/TCP   39s
```
30051 是系统随机分配的 nodePort，它和上面 NodePort 方式中的 nodePort 并无差别。集群内部可以使用 <clusterIP>:port 的方式访问服务，集群外也可以使用类似前述 <nodeIP>:nodePort ，但是，之所以使用 LoadBalancer，最主要的是可以使用 <externalIp>:port 的方式访问服务。externalIp 是由云供应商提供的负载均衡器的 IP 地址。
`port` is the port number which makes a service visible to other services running within the same k8s cluster. The port is usually used together with clusterIP of the service like <clusterIP>:port, cluserIP and port here are both virtual
`targetPort` is the port on the pods where the service's endpoints are running, the request will be load-balanced to one of the endpoint pods. `targetPort` should be keep same with the ports (container port) on the pods 
`nodePort` is the port on which the service can be accessed from externel using Kube-Proxy like <nodeIP>:nodePort. If `nodePort` is not sepcified, a random port will be used (preferred)

- spec.selector 
  选定 service 后端的一组 pod
  不带 selector 的 service 通常使用其它类型的后端（backend），例如
  ```
  kind: Service
  apiVersion: v1
  metadata:
  name: my-service
  spec:
	ports:
	  - protocol: TCP
	    port: 80
	    targetPort: 9376
  ```
  ```
  kind: Endpoints
  apiVersion: v1
  metadata:
    name: my-service
  subsets:
    - addresses:
      - ip: 10.2.3.4
    ports:
      - port: 9376
  ```
  访问 my-service 会被路由到  10.2.3.4:9376 上
  
cluster IP will be used by Service proxies. 每一个 node 上都运行着 kube-proxy 进程，它监控 kube master 进程对 Service 对象和 Endpoints 对象的添加和移除，并将 service-name, clusterIP, endpoint 的关系维护进 node 的 iptables 规则表（可以在 node 上使用 iptables-save 命令查看已经存在的关系表）；同时，这些更新也会被记录到 etcd 中。

为了实现服务发现，最开始的时候，kubernetes 采用了 docker 使用过的方法——环境变量。每个 pod 启动的时候，会将所有服务和 IP 及端口的对应关系初始化到 pod 的环境变量中，这样 pod 中的应用可以通过读取环境变量来获取其依赖的服务的地址信息。这种方式要求 pod 依赖的服务必须在 pod 启动之前就存在。
更理想的方案是应用直接使用服务的名字来访问服务，至于服务对应的 IP 则由 DNS 系统自动完成，Kubernetes 可以采用插件式的 DNS 实现该功能。这个 DNS 作为集群上运行的一个特殊应用存在。启动 kubelet 进程时，为其指定 DNS 服务的 cluster IP，那么与 kubelet 进程相同节点的 pod 用 service name 访问服务时，就会通过 kubelet 进程获取到相应服务的 cluster IP，进而请求被负载均衡到后端的 pod 上。

