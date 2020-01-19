---
title: 将你的应用部署在 K8S 上
date: 2020-01-16 18:36:35
categories: DevOps
tags:
    - k8s
    - docker
---
Kubernetes（简称 K8S）自发布以来，已经成为容器化编排的最佳工具，本文通过一个简单的案例聊一聊如何将一个容器部署到 k8s 集群上，文中所需的代码和容器均在 [GITHUB](https://github.com/AkisAya/simple-web-for-k8s-demo) 上。如需要搭建一个 k8s 集群，可以通过 [minikube](https://github.com/kubernetes/minikube) 来搭建，也可以使用 docker for desktop 自带的 k8s 集群

## 1 创建一个最简单的 Deployment
先简单介绍几个概念
<!-- more -->
- Pod: k8s 最核心的一个概念，是一个或多个 container 的集合。一个 pod 中的 container 共享存储和网络，k8s 支持多种 contianer，但是目前 docker container 是最常见的一种。我们的应用最终都是各个 pod 的形式存在的，pod 也是 k8s 进行调度的最小单位。
![pod](pod.png)
- ReplicaSet: 也是 k8s 中的一种资源，管理一个 pod 的多个副本。一般我们部署 pod 为了保证高可用，都会部署多个副本，避免单点故障。如果 replicaSet 管理的某个 pod 挂掉了，它会请求一个新的 pod 出来以满足设定的副本数量。k8s 还有一个 ReplicaionController 也是控制 pod 副本的，但是相比 ReplicaionController，ReplicaSet 对 pod 标签的支持会更好，现在一般就用 ReplicaSet
![replica](replica.png)
- Deployment: 是一个更 high-level 的资源，创建一个 Deployment 后会创建一个 ReplicaSet，然后 ReplicaSet 会创建预期数量的 pod。deployment 可以对 ReplicaSet 的版本进行管理，方便回滚。在生产上一般都是创建一个 Deployment
![depolyment](deployment.png)

我们先准备一个简单的 web 应用 simple-web，并将其打包成 docker 镜像 simple-web:1.0。web 应用后，访问 `/index`，将会输出 web 应用的名字，访问 `/health`，会返回 "OK"
```
> curl http://ip:8080/index
  web site name is [demo-web]
> curl http://ip:8080/health
  OK
```

定义一个 `deployment.yml` 如下
![config](deploy-config.png)
使用 `kubectl create -f deployment.yml`，然后 get 资源可以发现我们创建了一个 deployment，一个 replicaset，一个 pod
```
➜  ~ k get deploy
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
simple-web   1/1     1            1           2d4h
➜  ~ k get rs
NAME                    DESIRED   CURRENT   READY   AGE
simple-web-6bdf7b4d84   1         1         1       47h
➜  ~ k get pods
NAME                          READY   STATUS    RESTARTS   AGE
simple-web-6bdf7b4d84-p7jkp   1/1     Running   0          47h
```
我们可以进入到 pod 中执行 curl 命令查看输出，也可以把 curl 命令传递给 kubectl exec
```
➜  ~ k exec  simple-web-6bdf7b4d84-p7jkp --  curl -s http://localhost:8080/index
web site name is [demo-web]
```

## 2 使用 service 访问 pod
### 2.1 port-forward 
仅仅是创建一个 Deployment，我们还只能在 pod 内访问自己的 web 服务，而无法在外部或者 pod 之间访问 web 服务。`kubectl port-forward <podname> <hostPort:containerPort>` 可以将宿主机的一个 port 代理到容器的一个 pod 上，使我们在宿主机上访问 web 服务
```
➜  ~ k port-forward simple-web-6bdf7b4d84-p7jkp 8889:8080
Forwarding from 127.0.0.1:8889 -> 8080
Forwarding from [::1]:8889 -> 8080

➜  ~ curl 127.0.0.1:8889/index
web site name is [demo-web]
```
### 2.2 service
但是当我们有多个副本的 pod 时，我们不可能一个一个 pod 的代理指定端口，同时 port-forward 也没解决 pod 之间的服务发现。这个时候我们需要 `Service` 来提供路由
![service](service.png)
我们同样使用 ``kubectl create` 创建一个 `Service`
```
apiVersion: v1
kind: Service
metadata:
  name: simple-web
spec:
  selector:
    app: simple-web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```
这里 selector 我们需要填写在 deployment 中定义的 pod 的标签
```
➜  ~ k get svc
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
simple-web            ClusterIP   10.96.156.194   <none>        80/TCP         2d4h
```
我们创建了一个叫 `simple-web` 的 service，可以看到 k8s 分配了一个 clusterIP，并将这个 ip 的 80 端口映射到 pod 的 8080 端口
```
➜  ~ curl 10.96.156.194/index
web site name is [demo-web]
```
除了用 ip 访问，我们也可以用 serivce 来访问
```
➜  ~ k exec simple-web-6bdf7b4d84-2kzxx -- curl http://simple-web/index
web site name is [demo-web]
```
### 2.3 nodeport
使用 ClusterIP 我们只能在 k8s 集群内访问 pod，如果要从外部访问 pod，则可以定义 NodePort type 的 Service
```
apiVersion: v1
kind: Service
metadata:
  name: simple-web
spec:
  type: NodePort
  selector:
    app: simple-web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 32100
```
这样我们就可以通过宿主机 ip:nodeport 访问 pod 中的 web 服务了
Service 还提供了 LoadBalancer 的 type，方便提供一个 external 的 ip 供访问，一般为云厂商使用，此处不表

## 3 Volume
Volume 提供了一种在 pod 的 container 之间数据交互，container 与宿主机或者其它的存储交互的手段。k8s 的 Volume 有很多类型，下面以 EmptyDir 为例看看 Volume 的使用。
我们将 web 应用的 log 挂载到 EmptyDir，配置如下
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: simple-web
  template:
    metadata:
      labels:
        app: simple-web
    spec:
      containers:
        - name: simple-web
          image: simple-web:1.0
          volumeMounts:
            - name: app-log
              mountPath: /app/logs
      volumes:
        - name: app-log
          emptyDir: {}
```
我们可以在宿主机上找到 emptyDir 的路径，默认是 `/var/lib/kubelet/pods/PODUID/volumes/kubernetes.io~empty-dir/VOLUMENAME`
```
➜  ~ k get pods simple-web-6bdf7b4d84-2kzxx -o yaml | grep uid
    uid: 437573bc-7398-4004-89ac-0d268e986c74
  uid: be609483-ed6d-427f-b277-81931cc9eb60
```
`be609483-ed6d-427f-b277-81931cc9eb60` 就是 pod uid
以这个 log 为例，我们宿主机上的 log 可以通过 filebeat 收集起来

## 4 configmap
当我们把应用部署到 pod 后，如果希望通过更改配置更改应用的行为，肯定不希望是需要重新打包一个镜像，所以我们需要把配置文件分离出来，configmap 就可以帮助我们完成这个事
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: simple-web-config
data:
  app-config: |-
    app.name=web-name-from-config-web
```
然后修改 deployment 如下
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: simple-web
  template:
    metadata:
      labels:
        app: simple-web
    spec:
      containers:
        - name: simple-web
          image: simple-web:1.0
          volumeMounts:
            - name: app-log-vol
              mountPath: /app/logs
            - name: config-vol
              mountPath: /app/conf/application.properties
              subPath: application.properties
      volumes:
        - name: app-log-vol
          emptyDir: {}
        - name: config-vol
          configMap:
            name: simple-web-config
            items:
              - key: app-config
                path: application.properties
```
我们把 configmap 里的配置挂载到了 `/app/conf/application.properties`，我们应用的启动脚本会读取这个配置文件 `java -jar simple-web.jar --spring.config.location=/app/conf/application.properties`，通过这种方式，我们就将配置文件与应用分离。修改配置文件后，`application.properties` 就已经被更新了，但是由于 java 应用不会重新加载配置文件，所以需要重启 pod

## 5 liveness, readiness 与 pod resources
### 5.1 liveness, readiness
k8s 是如何知道一个 pod 无法提供服务了呢？我们需要定义一个健康检查的 liveness ，k8s 会定期请求这个 liveness，判断服务是否健康。
readiness 类似，通过 readiness 的定义判读 pod 是否已经可以开始提供服务
### 5.2 resources
k8s 提供了限制 container 所能使用的资源量的方法。`resources.requests` 定义了需要多少资源，用于 k8s 将 pod 调度到资源充足的节点，`resources.limits` 则限制了 container 可以使用的资源总量。可参考[Kubernetes Container Resource Requirements](https://medium.com/expedia-group-tech/kubernetes-container-resource-requirements-part-1-memory-a9fbe02c8a5f)
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: simple-web
  template:
    metadata:
      labels:
        app: simple-web
    spec:
      containers:
        - name: simple-web
          image: simple-web:1.0
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
          volumeMounts:
            - name: app-log-vol
              mountPath: /app/logs
            - name: config-vol
              mountPath: /app/conf/application.properties
              subPath: application.properties
          resources:
              requests:
                memory: 2048Mi
                cpu: 2
              limits:
                memory: 2048Mi
                cpu: 2
      volumes:
        - name: app-log-vol
          emptyDir: {}
        - name: config-vol
          configMap:
            name: simple-web-config
            items:
              - key: app-config
                path: application.properties
```

## 6 其它待补充，pause 容器等，dashboard