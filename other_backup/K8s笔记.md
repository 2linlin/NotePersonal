### 1.创建集群

K8s包含两类类型的节点：

- Master：负责整个集群的调度。
- Node：负责运行应用。每个Node都有 Kubelet , 它管理 Node 而且是 Node 与 Master 通信的代理。生产机的K8s至少要有3个Node。

Node和终端用户都可用Master暴露的API和它通信。K8s是C/S模式。

Minikube 是一种轻量级的 K8s 实现，可以用它来练手。

实操：

```shell
# 查看是否已安装minikube
minikube version
# 启动集群
minikube start
# 查看kubectl客户端是否已安装
kubectl version
# 查看集群信息（mastar,DNS信息）
kubectl cluster-info
# 查看节点列表
kubectl get nodes
```

### 2.部署应用

部署应用需要创建k8s `Deployment`，它用来指挥K8s创建、更新应用程序实例。

应用程序创建后，k8s Deployment控制器会持续监控这些实例，并提供自动修复机制。

后续也可以通过更新Deployment来更新应用程序的部署。Deployment是在Master上。

实操：

```shell
# 发布应用
kubectl create deployment k8s-app1 --image=gcr.io/google-samples/kubernetes-bootcamp:v1
# 查看发布列表
kubectl get deployments
# 打开proxy代理以进入k8s内网（一般是新开一个Terminal打开），相当于本地网络映射到k8s内网
kubectl proxy
# 直接使用k8s API访问k8s功能（需要先打开代理），下面的8081是k8s代理在本地的映射
curl http://localhost:8001/version
# 打开代理后还可以通过pod的name直接访问pod
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```

如果想不打开代理就访问K8s内网，那么我们就用到了之后要讲的Service。

### 3.查看应用信息

各术语粒度：Node -> Deployment -> Pod

常见的操作可以用以下命令完成：

```shell
kubectl get # 列出资源
kubectl describe # 显示有关资源的详细信息
kubectl logs # 打印 pod 和其中容器的日志
kubectl exec # 在 pod 中的容器上执行命令
```

实操：

```shell
# 查看所有pod信息
kubectl describe pods

# 打开代理
echo -e "\n\n\n\e[92mStarting Proxy. After starting it will not output a response. Please click the first Terminal Tab\n"; 
kubectl proxy
# 获取Pod name
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
# 直接用k8s的API通过代理、pod name访问pod
curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME/proxy/

# 查看Pod日志
kubectl logs $POD_NAME

# 列出pod的环境变量
kubectl exec $POD_NAME env

# 直接进入pod
kubectl exec -ti $POD_NAME bash
```

### 4.用Service暴露应用

每个k8s的pod都有一个独立的IP，甚至是同一个node上的pod的IP也是不一样的。

一个service是对一套pod的抽象，然后提供了访问它们的策略。Service和其他的k8s对象一样，通过Yaml（推荐）或json来定义。

Service可用通过以下几种方式来暴露：

- ClusterIP (default)：默认的方式。这种方式，Service只能在集群内部访问到。
- NodePort：在每个Node上用相同的port暴露。可在外部用`<NodeIP>:<NodePort>`访问，ClusterIP的超集。
- LoadBalancer：负载均衡方式，提供可维护的IP，NodePort的超集。
- ExternalName：用制定的名字暴露，kube-dns版本1.7及以上支持。

Service通过label和selector来匹配Pod集。通过Key/Value可随时进行编辑。

实操：

```shell
# =====创建新Service
# 查看service列表
kubectl get services
# 创建新service，并暴露（NodePort方式）
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
# 列出service信息（哪个port暴露等等）
kubectl describe services/kubernetes-bootcamp
# 把暴露的端口存到环境变量
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
# 直接用外部暴露的NodePort访问k8s
curl $(minikube ip):$NODE_PORT
```

```shell
# =====使用labels
# 前面的deployment为我们自动创建了label，我们可以看一下
kubectl describe deployment
...
Labels:                 run=kubernetes-bootcamp
...

# 使用label查询pods，加-l参数即可
kubectl get pods -l run=kubernetes-bootcamp

# 也可以用同样的方式查询service
kubectl get services -l run=kubernetes-bootcamp

# 创建新label，用“lable + 对象类型 + 对象名字 + 新label”即可
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
kubectl label pod $POD_NAME app=v1
# 用上述同样的方式就可以搜索到pod
kubectl get pods -l app=v1
```

```shell
# =====删除一个Service
# 通过label来过滤删除service
kubectl delete service -l run=kubernetes-bootcamp

# 通过外网检查下，已经访问不到pod了
curl $(minikube ip):$NODE_PORT

# 通过内网还能访问，因为删除的是Service，但是Deployment还在管理pod
kubectl exec -ti $POD_NAME curl localhost:8080
```

### 5.扩容缩容

通过Deployment来实现。

```shell
# 查看已有的deployments
kubectl get deployments
# 查看所有deployments创建的pod
kubectl get rs

# =======扩容
# 将deployments规模扩充到4个
kubectl scale deployments/kubernetes-bootcamp --replicas=4
# 看一个，pod已经扩充到4个了
kubectl get deployments
kubectl get pods -o wide
# 扩容的操作被记录到了日志里，可以看下
kubectl describe deployments/kubernetes-bootcamp

# =======负载均衡
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
# 我们可以看下，负载均衡已经生效了，每次访问都是不同的pod
curl $(minikube ip):$NODE_PORT

# =======缩容
# 和扩容同样的操作
kubectl scale deployments/kubernetes-bootcamp --replicas=2
kubectl get deployments
kubectl get pods -o wide
```

### 6.滚动更新

默认不可用pod最大值是1，可创建的新pod也是1。这两个数字可以配置成数量或百分比。

k8s的更新都经过了版本控制，支持回滚。

与应用程序扩展类似，如果公开了 Deployment，服务将在更新期间仅对可用的 pod 进行负载均衡。可用 Pod 是应用程序用户可用的实例。

滚动更新支持以下操作：

- 将应用程序从一个环境提升到另一个环境（通过容器镜像更新）
- 回滚到以前的版本
- 持续集成和持续交付应用程序，无需停机

实操：

```shell
# =====更新app版本
# 查看deployments列表
kubectl get deployments
# 查看pods列表
kubectl get pods
# 查看镜像版本
kubectl describe pods
# 更新镜像
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
# 更新镜像后pod就会自动重启
kubectl get pods

# =====验证已更新
# 手动验证，可以看到版本已经更新了
kubectl describe services/kubernetes-bootcamp
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
curl $(minikube ip):$NODE_PORT
# 直接验证
kubectl rollout status deployments/kubernetes-bootcamp
kubectl describe pods

# =====回滚更新
# 升级
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=gcr.io/google-samples/kubernetes-bootcamp:v10
kubectl get deployments
kubectl get pods
kubectl describe pods
# 回滚
kubectl rollout undo deployments/kubernetes-bootcamp
kubectl get pods
kubectl describe pods
```

## 常用命令

```
alias kc='kubectl' alias kp='kubectl get po' alias kpa='kubectl get po --all-namespaces' alias kd='kubectl describe ' alias kv='kubectl get svc' alias kf='kubectl apply -f ' alias kdf='kubectl delete -f ' alias kdp='kubectl delete po ' alias kl='kubectl logs ' alias kg='kubectl get '
```

创建、删除namespace

```shell
kubectl get namespaces
# 创建namespace，命令行方式
kubectl create namespace new-namespace
# 创建namespace，文件方式
$ cat my-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: new-namespace
$ kubectl create -f ./my-namespace.yaml
#删除namespace
$ kubectl delete namespaces new-namespace
```

