## 内网kubernetes集群基于traefik对外提供服务

### 一、前置知识点

**理解Ingress**

简单的说，ingress就是从kubernetes集群外访问集群的入口，将用户的URL请求转发到不同的service上。Ingress相当于nginx、apache等负载均衡方向代理服务器，其中还包括规则定义，即URL的路由信息，路由信息得的刷新由[Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers)来提供。

**理解Ingress Controller**

Ingress Controller 实质上可以理解为是个监视器，Ingress Controller 通过不断地跟 kubernetes API 打交道，实时的感知后端 service、pod 等变化，比如新增和减少 pod，service 增加与减少等；当得到这些变化信息后，Ingress Controller 再结合下文的 Ingress 生成配置，然后更新反向代理负载均衡器，并刷新其配置，达到服务发现的作用。

![image-20210511141608329](D:\MyData\ex_zhongyq\AppData\Roaming\Typora\typora-user-images\image-20210511141608329.png)

**理解kubernetes-控制器Deployment和DaemonSet**

- **Deployment**

Deployment为Pod和Replica Set（下一代Replication Controller）提供声明式更新。

只需要在 Deployment 中描述想要的目标状态是什么，Deployment controller 就会帮您将 Pod 和ReplicaSet 的实际状态改变到您的目标状态。也可以定义一个全新的 Deployment 来创建 ReplicaSet 或者删除已有的 Deployment 并创建一个新的来替换。

![img](https://img2018.cnblogs.com/blog/1156961/201812/1156961-20181224103448131-1363236292.png)



- **DaemonSet**

DaemonSet 确保全部（或者一些）Node 上运行一个 Pod 的副本。当有 Node 加入集群时，也会为他们新增一个 Pod 。当有 Node 从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。

使用 DaemonSet 的一些典型用法：

```
1、运行集群存储 daemon，例如在每个 Node 上运行 ，应用场景：Agent。
2、在每个 Node 上运行日志收集 daemon，例如fluentd、logstash。
3、在每个 Node 上运行监控 daemon，例如 Prometheus Node Exporter、collectd、Datadog 代理、New Relic 代理，或 Ganglia gmond。
```



### 二、traefik 简介



![traefik-architecture](D:\MyData\ex_zhongyq\Pictures\traefik-architecture.png)



[**Traefik**](https://traefik.io/)是一款开源的反向代理与负载均衡工具。它最大的优点是能够与常见的微服务系统直接整合，可以实现自动化动态配置。目前支持Docker, Swarm, Mesos/Marathon, Mesos, Kubernetes, Consul, Etcd, Zookeeper, BoltDB, Rest API等等后端模型。至于使用它的原因则基于以下几点：

- 无须重启即可更新配置
- 自动的服务发现与负载均衡
- 与 `docker` 的完美集成，基于 `container label` 的配置
- 漂亮的 `dashboard` 界面
- `metrics` 的支持，对 `prometheus` 和 `k8s` 的集成



**部署方式**

- **traefik部署在k8s上分为daemonset和deployment两种方式各有优缺点：**

daemonset 能确定有哪些node在运行traefik，所以可以确定的知道后端ip，但是不能方便的伸缩

deployment 可以更方便的伸缩，但是不能确定有哪些node在运行traefik所以不能确定的知道后端ip

一般部署两种不同类型的traefik:

面向内部(internal)服务的traefik，建议可以使用deployment的方式
面向外部(external)服务的traefik，建议可以使用daemonset的方式

本文使用的是daemonset方式部署。



### 三、部署Traefik

**环境介绍**

| 操作系统   | ip              | 主机名     | 配置  | 备注              |
| ---------- | --------------- | ---------- | ----- | ----------------- |
| centos 7.6 | 192.168.241.253 | k8s-master | 2核4G | Kubernetesv1.20.5 |
| centos 7.6 | 192.168.241.132 | k8s-node01 | 2核8G | Kubernetesv1.20.5 |
| centos 7.6 | 192.168.241.135 | k8s-node02 | 2核8G | v1.20.5           |
| centos 7.6 | 192.168.241.137 | k8s-node03 | 2核8G | v1.20.5           |



**目录结构如下**

```
traefik-v2.1.6
├── traefik-config.yaml
├── traefik-ds-v2.1.6.yaml
├── traefik-rbac.yaml
└── ui.yaml
```

**traefik-config.yaml**

```
# cat traefik-config.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: traefik-config
  namespace: kube-system
data:
  traefik.toml: |
    defaultEntryPoints = ["http","https"]
    debug = false
    logLevel = "INFO"
    # Do not verify backend certificates (use https backends)
    InsecureSkipVerify = true
    [entryPoints]
      [entryPoints.http]
      address = ":80"
      compress = true
      [entryPoints.https]
      address = ":443"
        [entryPoints.https.tls]
    #Config to redirect http to https
    #[entryPoints]
    #  [entryPoints.http]
    #  address = ":80"
    #  compress = true
    #    [entryPoints.http.redirect]
    #    entryPoint = "https"
    #  [entryPoints.https]
    #  address = ":443"
    #    [entryPoints.https.tls]
    [web]
      address = ":8080"
    [kubernetes]
    [metrics]
      [metrics.prometheus]
      buckets=[0.1,0.3,1.2,5.0]
      entryPoint = "traefik"
    [ping]
    entryPoint = "http"
```

**traefik-ds-v2.1.6.yaml**

```
# cat traefik-ds-v2.1.6.yaml

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: DaemonSet
apiVersion: apps/v1
#apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller-v2
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  selector:
    matchLabels:
      name: traefik-ingress-lb-v2
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb-v2
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik:2.1.6
        name: traefik-ingress-lb-v2
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: admin
          containerPort: 8080
          hostPort: 8080
        securityContext:
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        args:
        - --api
        - --api.insecure=true
        - --providers.kubernetesingress=true
        - --log.level=INFO
       # - --configfile=/config/traefik.toml
       # volumeMounts:
       # - mountPath: /config
       #   name: config
      volumes:
      - configMap:
          name: traefik-config
        name: config
---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service-v2
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb-v2
spec:
  selector:
    k8s-app: traefik-ingress-lb-v2
  ports:
    - protocol: TCP
      port: 80
      name: web
    - protocol: TCP
      port: 8080
      name: admin
```

**traefik-rbac.yaml**

```
# cat traefik-rbac.yaml

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
    - extensions
    resources:
    - ingresses/status
    verbs:
    - update
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
```

**ui.yaml**

```
# cat ui.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - name: web
    port: 80
    targetPort: 8080
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  rules:
  - host: prod-traefik-ui.bgbiao.cn       //域名访问
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-web-ui
          servicePort: web
```



**部署**

```
kubectl apply -f .
```

**查看pod**

```
# kubectl get pods -n kube-system | grep traefik
traefik-ingress-controller-v2-qglz7        1/1     Running   0          6h41m
traefik-ingress-controller-v2-spgkx        1/1     Running   0          6h41m
traefik-ingress-controller-v2-xgg4g        1/1     Running   0          6h41m
```

 **查看svc**

```
# kubectl get svc -n kube-system | grep traefik
traefik-ingress-service-v2   ClusterIP   10.233.48.148   <none>        80/TCP,8080/TCP          6h42m
traefik-web-ui               ClusterIP   10.233.38.93    <none>        80/TCP                   6h42m

```

**查看ingresses**

```
# kubectl get ingresses.extensions -n kube-system
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
NAME             CLASS    HOSTS                       ADDRESS   PORTS   AGE
traefik-web-ui   <none>   prod-traefik-ui.bgbiao.cn             80      6h42m

```

**域名访问**

```
浏览器中输入
prod-traefik-ui.bgbiao.cn

如果没有域名请在本地添加hosts解析
ip prod-traefik-ui.bgbiao.cn
```

**使用ip访问**

```
nodeIP:8080
http://192.168.241.132:8080/
```

效果如下：

![image-20210511150208030](D:\MyData\ex_zhongyq\AppData\Roaming\Typora\typora-user-images\image-20210511150208030.png)