# 第二十二天：k8s基礎篇延伸與EKS

Author: Nick Zhuang
Type: kubernetes

# 前言

昨天我們已經把k8s的集群在AWS上架設起來了，今天我們稍微操作下基礎篇的內容，看看會有什麼差異，值得一提的是，我們在昨天新增了三個Node，這三個Node是計算節點，主節點Master的部分是EKS在管理的，我們不能去操控它，並且它自帶HA（High Availability），高可用性，也就是有主節點的備援，我們不需要再集群內新增主節點並設置HA，HA的好處是當主節點掛掉的時候，它會用另外兩個備用的選一個當作新的主節點，並會找一個新的主備用節點，維持同時三個主節點的狀態。k8s在使用上是基於etcd的，它是一個時間序列的資料庫，因此備援系統的設置顯得格外重要，它可以保證群集資料的完整性。

# 基礎篇

接著我們應用前面所學的知識，操作基礎篇的內容

在開始之前，我們先確定版本

    $kubectl version --short
    Client Version: v1.13.11-eks-5876d6
    Server Version: v1.13.10-eks-5ac0f1

## 示例一：有標籤和狀態的Pod

我們前面有提過標籤（Label）、活性（livenessProbe）、可讀性（readinessProbe）

- label：標示用的，可以用來搜尋篩選用，如果只是單純註解，可以用annotation
- liveness probe：針對pod的自我健康狀態檢查
- readiness probe：針對pod的可access的檢查

在這邊我們綜合運用一下

    $vim pod-demo.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-helloworld
      labels:
        app: helloworld
    spec:
      containers:
      - name: k8s-demo
        image: 105552010/k8s-demo
        args:
        - /bin/sh
        - -c
        - touch /tmp/healthy; npm start; sleep 30; rm -rf /tmp/healthy; sleep 600
        ports:
        - name: nodejs-port
          containerPort: 3000
        livenessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 15
          timeoutSeconds: 30
        readinessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 5
          periodSeconds: 5

接著apply它並檢查Pod狀態

    $kubectl apply -f pod-demo.yaml
    pod/my-helloworld created
    $kubectl get po
    NAME            READY   STATUS    RESTARTS   AGE
    my-helloworld   1/1     Running   0          20s

詳細檢查，可以發現到不一樣

    $kubectl describe po
    Name:               my-helloworld
    Namespace:          default
    Priority:           0
    PriorityClassName:  <none>
    Node:               ip-192-168-18-184.ap-southeast-1.compute.internal/192.168.18.184
    Start Time:         Mon, 07 Oct 2019 16:09:42 +0800
    Labels:             app=helloworld
    Annotations:        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"app":"helloworld"},"name":"my-helloworld","namespace":"default"}
                        kubernetes.io/psp: eks.privileged
    Status:             Running
    IP:                 192.168.28.191
    Containers:
      k8s-demo:
        Container ID:  docker://ea5c9229dd6589a136dfd72aafa63333081f4e31403a16325634f27517f359bc
        Image:         105552010/k8s-demo
        Image ID:      docker-pullable://105552010/k8s-demo@sha256:2c050f462f5d0b3a6430e7869bcdfe6ac48a447a89da79a56d0ef61460c7ab9e
        Port:          3000/TCP
        Host Port:     0/TCP
        Args:
          /bin/sh
          -c
          touch /tmp/healthy; npm start; sleep 30; rm -rf /tmp/healthy; sleep 600
        State:          Running
          Started:      Mon, 07 Oct 2019 16:09:46 +0800
        Ready:          True
        Restart Count:  0
        Liveness:       http-get http://:3000/ delay=15s timeout=30s period=10s #success=1 #failure=3
        Readiness:      exec [cat /tmp/healthy] delay=5s timeout=1s period=5s #success=1 #failure=3
        Environment:    <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from default-token-b6shs (ro)
    Conditions:
      Type              Status
      Initialized       True
      Ready             True
      ContainersReady   True
      PodScheduled      True
    Volumes:
      default-token-b6shs:
        Type:        Secret (a volume populated by a Secret)
        SecretName:  default-token-b6shs
        Optional:    false
    QoS Class:       BestEffort
    Node-Selectors:  <none>
    Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                     node.kubernetes.io/unreachable:NoExecute for 300s
    Events:
      Type    Reason     Age    From                                                        Message
      ----    ------     ----   ----                                                        -------
      Normal  Scheduled  2m25s  default-scheduler                                           Successfully assigned default/my-helloworld to ip-192-168-18-184.ap-southeast-1.compute.internal
      Normal  Pulling    2m24s  kubelet, ip-192-168-18-184.ap-southeast-1.compute.internal  pulling image "105552010/k8s-demo"
      Normal  Pulled     2m21s  kubelet, ip-192-168-18-184.ap-southeast-1.compute.internal  Successfully pulled image "105552010/k8s-demo"
      Normal  Created    2m21s  kubelet, ip-192-168-18-184.ap-southeast-1.compute.internal  Created container
      Normal  Started    2m21s  kubelet, ip-192-168-18-184.ap-southeast-1.compute.internal  Started container

節點等訊息可以看出與minikube的差異

恢復原狀

    $kubectl delete -f pod-demo.yaml
    pod "my-helloworld" deleted

接著我們看下一個例子

## 示例二：Scaling

前面有提過Scaling的部分包含Horizontal和Vertical

- Vertical Scaling：水平拓展，能夠分配更多的系統資源給Pod
- Horizontal Scaling：垂直拓展，可以創建分身，但是必須是Stateless的Pod

我們看個例子，用ReplicationController測試

    $vim pod-scale.yaml
    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: helloworld-controller
    spec:
      replicas: 2
      selector:
        app: helloworld
      template:
        metadata:
          labels:
            app: helloworld
        spec:
          containers:
          - name: k8s-demo
            image: 105552010/k8s-demo
            ports:
            - containerPort: 3000
            resources:
              requests:
                cpu: 100m
                memory: 50Mi

接著一樣apply，很熟悉吧？

    $kubectl apply -f pod-scale.yaml
    replicationcontroller/helloworld-controller created

檢查Pod狀態

    $kubectl get po
    NAME                          READY   STATUS              RESTARTS   AGE
    helloworld-controller-9gb5m   0/1     ContainerCreating   0          20s
    helloworld-controller-bz9t2   1/1     Running             0          20s
    $kubectl get po
    NAME                          READY   STATUS    RESTARTS   AGE
    helloworld-controller-9gb5m   1/1     Running   0          23s
    helloworld-controller-bz9t2   1/1     Running   0          23s

詳細檢查Pod狀態

    $kubectl describe po
    Name:               helloworld-controller-9gb5m
    Namespace:          default
    Priority:           0
    PriorityClassName:  <none>
    Node:               ip-192-168-51-244.ap-southeast-1.compute.internal/192.168.51.244
    Start Time:         Mon, 07 Oct 2019 16:44:51 +0800
    Labels:             app=helloworld
    Annotations:        kubernetes.io/psp: eks.privileged
    Status:             Running
    IP:                 192.168.54.6
    Controlled By:      ReplicationController/helloworld-controller
    Containers:
      k8s-demo:
        Container ID:   docker://0292d73e55d870d549d31fe2f54320d3ad4af49d3d04aad42f0e6f7a0d729de7
        Image:          105552010/k8s-demo
        Image ID:       docker-pullable://105552010/k8s-demo@sha256:2c050f462f5d0b3a6430e7869bcdfe6ac48a447a89da79a56d0ef61460c7ab9e
        Port:           3000/TCP
        Host Port:      0/TCP
        State:          Running
          Started:      Mon, 07 Oct 2019 16:45:11 +0800
        Ready:          True
        Restart Count:  0
        Requests:
          cpu:        100m
          memory:     50Mi
        Environment:  <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from default-token-b6shs (ro)
    Conditions:
      Type              Status
      Initialized       True
      Ready             True
      ContainersReady   True
      PodScheduled      True
    Volumes:
      default-token-b6shs:
        Type:        Secret (a volume populated by a Secret)
        SecretName:  default-token-b6shs
        Optional:    false
    QoS Class:       Burstable
    Node-Selectors:  <none>
    Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                     node.kubernetes.io/unreachable:NoExecute for 300s
    Events:
      Type    Reason     Age   From                                                        Message
      ----    ------     ----  ----                                                        -------
      Normal  Scheduled  73s   default-scheduler                                           Successfully assigned default/helloworld-controller-9gb5m to ip-192-168-51-244.ap-southeast-1.compute.internal
      Normal  Pulling    72s   kubelet, ip-192-168-51-244.ap-southeast-1.compute.internal  pulling image "105552010/k8s-demo"
      Normal  Pulled     58s   kubelet, ip-192-168-51-244.ap-southeast-1.compute.internal  Successfully pulled image "105552010/k8s-demo"
      Normal  Created    53s   kubelet, ip-192-168-51-244.ap-southeast-1.compute.internal  Created container
      Normal  Started    53s   kubelet, ip-192-168-51-244.ap-southeast-1.compute.internal  Started container
    
    
    Name:               helloworld-controller-bz9t2
    Namespace:          default
    Priority:           0
    PriorityClassName:  <none>
    Node:               ip-192-168-18-184.ap-southeast-1.compute.internal/192.168.18.184
    Start Time:         Mon, 07 Oct 2019 16:44:51 +0800
    Labels:             app=helloworld
    Annotations:        kubernetes.io/psp: eks.privileged
    Status:             Running
    IP:                 192.168.14.232
    Controlled By:      ReplicationController/helloworld-controller
    Containers:
      k8s-demo:
        Container ID:   docker://4c5434332b92fef28fbdbffa69738ad13f7a960d42c5ea147262b55fbf923e8b
        Image:          105552010/k8s-demo
        Image ID:       docker-pullable://105552010/k8s-demo@sha256:2c050f462f5d0b3a6430e7869bcdfe6ac48a447a89da79a56d0ef61460c7ab9e
        Port:           3000/TCP
        Host Port:      0/TCP
        State:          Running
          Started:      Mon, 07 Oct 2019 16:44:55 +0800
        Ready:          True
        Restart Count:  0
        Requests:
          cpu:        100m
          memory:     50Mi
        Environment:  <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from default-token-b6shs (ro)
    Conditions:
      Type              Status
      Initialized       True
      Ready             True
      ContainersReady   True
      PodScheduled      True
    Volumes:
      default-token-b6shs:
        Type:        Secret (a volume populated by a Secret)
        SecretName:  default-token-b6shs
        Optional:    false
    QoS Class:       Burstable
    Node-Selectors:  <none>
    Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                     node.kubernetes.io/unreachable:NoExecute for 300s
    Events:
      Type    Reason     Age   From                                                        Message
      ----    ------     ----  ----                                                        -------
      Normal  Scheduled  73s   default-scheduler                                           Successfully assigned default/helloworld-controller-bz9t2 to ip-192-168-18-184.ap-southeast-1.compute.internal
      Normal  Pulling    72s   kubelet, ip-192-168-18-184.ap-southeast-1.compute.internal  pulling image "105552010/k8s-demo"
      Normal  Pulled     69s   kubelet, ip-192-168-18-184.ap-southeast-1.compute.internal  Successfully pulled image "105552010/k8s-demo"
      Normal  Created    69s   kubelet, ip-192-168-18-184.ap-southeast-1.compute.internal  Created container
      Normal  Started    69s   kubelet, ip-192-168-18-184.ap-southeast-1.compute.internal  Started container

可以注意到它把Pod開在兩個不同的EC2的instance上

恢復原狀

    $kubectl delete -f pod-scale.yaml
    replicationcontroller "helloworld-controller" deleted

當然，我們也知道這可以用Deployment的ReplicaSet代替

    $vim pod-deploy.yaml
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: helloworld-deployment
    spec:
      replicas: 2
      template:
        metadata:
          labels:
            app: helloworld
        spec:
          containers:
          - name: k8s-demo
            image: 105552010/k8s-demo
            ports:
            - containerPort: 3000
            resources:
              requests:
                cpu: 100m
                memory: 50Mi

接著建立它

    $kubectl create -f pod-deploy.yaml
    deployment.extensions/helloworld-deployment created

檢查狀態，這個Deployment開啟了2個Pod，每個Pod的要求資源是cpu：100m、memory：50Mi

    $kubectl get deployment
    NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
    helloworld-deployment   2/2     2            2           3m55s
    $kubectl describe deploy
    Name:                   helloworld-deployment
    Namespace:              default
    CreationTimestamp:      Mon, 07 Oct 2019 17:02:09 +0800
    Labels:                 app=helloworld
    Annotations:            deployment.kubernetes.io/revision: 1
    Selector:               app=helloworld
    Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
    StrategyType:           RollingUpdate
    MinReadySeconds:        0
    RollingUpdateStrategy:  1 max unavailable, 1 max surge
    Pod Template:
      Labels:  app=helloworld
      Containers:
       k8s-demo:
        Image:      105552010/k8s-demo
        Port:       3000/TCP
        Host Port:  0/TCP
        Requests:
          cpu:        100m
          memory:     50Mi
        Environment:  <none>
        Mounts:       <none>
      Volumes:        <none>
    Conditions:
      Type           Status  Reason
      ----           ------  ------
      Available      True    MinimumReplicasAvailable
    OldReplicaSets:  <none>
    NewReplicaSet:   helloworld-deployment-5b5457694d (2/2 replicas created)
    Events:
      Type    Reason             Age   From                   Message
      ----    ------             ----  ----                   -------
      Normal  ScalingReplicaSet  4m6s  deployment-controller  Scaled up replica set helloworld-deployment-5b5457694d to 2

我們接著測試一些東西：

### Update（更新，升版）

將deployment設成Pause狀態

    $kubectl rollout pause deployment/helloworld-deployment
    deployment.extensions/helloworld-deployment paused

修改Deployment可使用的資源

    $kubectl set resources deployment helloworld-deployment --limits=cpu=200m,memory=100Mi
    deployment.extensions/helloworld-deployment resource requirements updated

更新所用的image版本

    $kubectl set image deployment/helloworld-deployment k8s-demo=105552010/k8s-demo:v2
    deployment.extensions/helloworld-deployment image updated

回復正常狀態

    $kubectl rollout resume deploy/helloworld-deployment
    deployment.extensions/helloworld-deployment resumed

檢查Pod狀態

    $kubectl get po
    NAME                                     READY   STATUS              RESTARTS   AGE
    helloworld-deployment-5b5457694d-hbw78   1/1     Running             0          12m
    helloworld-deployment-5b5457694d-v95bg   1/1     Terminating         0          12m
    helloworld-deployment-9d595467-s4z9p     0/1     ContainerCreating   0          6s
    helloworld-deployment-9d595467-s7wfx     0/1     ContainerCreating   0          6s
    $kubectl get po
    NAME                                     READY   STATUS        RESTARTS   AGE
    helloworld-deployment-5b5457694d-hbw78   0/1     Terminating   0          13m
    helloworld-deployment-9d595467-s4z9p     1/1     Running       0          40s
    helloworld-deployment-9d595467-s7wfx     1/1     Running       0          40s
    $kubectl get po
    NAME                                   READY   STATUS    RESTARTS   AGE
    helloworld-deployment-9d595467-s4z9p   1/1     Running   0          64s
    helloworld-deployment-9d595467-s7wfx   1/1     Running   0          64s

檢查Image版本

    $kubectl describe deployment|grep -i image
        Image:      105552010/k8s-demo:v2

創造更多分身（從2改成3）

    $kubectl scale deployment/helloworld-deployment --replicas=3
    deployment.extensions/helloworld-deployment scaled

檢查Pod狀態

    $kubectl get po
    NAME                                   READY   STATUS              RESTARTS   AGE
    helloworld-deployment-9d595467-dkxg4   0/1     ContainerCreating   0          19s
    helloworld-deployment-9d595467-s4z9p   1/1     Running             0          5m32s
    helloworld-deployment-9d595467-s7wfx   1/1     Running             0          5m32s
    $kubectl get po
    NAME                                   READY   STATUS    RESTARTS   AGE
    helloworld-deployment-9d595467-dkxg4   1/1     Running   0          56s
    helloworld-deployment-9d595467-s4z9p   1/1     Running   0          6m9s
    helloworld-deployment-9d595467-s7wfx   1/1     Running   0          6m9s

我們也可以設置AutoScaling

這邊是說最小4個Pod，最多10個，當cpu的loading大於80%就開始增加Pod，後面會詳細介紹

    $kubectl autoscale deployment/helloworld-deployment --min=4 --max=10 --cpu-percent=80
    horizontalpodautoscaler.autoscaling/helloworld-deployment autoscaled

檢查Pod狀態，最小是4，所以至少要4個

    $kubectl get po
    NAME                                   READY   STATUS    RESTARTS   AGE
    helloworld-deployment-9d595467-dkxg4   1/1     Running   0          3m52s
    helloworld-deployment-9d595467-phvvr   1/1     Running   0          76s
    helloworld-deployment-9d595467-s4z9p   1/1     Running   0          9m5s
    helloworld-deployment-9d595467-s7wfx   1/1     Running   0          9m5s

檢查更新狀態

    $kubectl rollout status deployment/helloworld-deployment
    deployment "helloworld-deployment" successfully rolled out

檢查詳細記錄

    $kubectl describe deployment
    Name:                   helloworld-deployment
    Namespace:              default
    CreationTimestamp:      Mon, 07 Oct 2019 17:02:09 +0800
    Labels:                 app=helloworld
    Annotations:            deployment.kubernetes.io/revision: 2
    Selector:               app=helloworld
    Replicas:               4 desired | 4 updated | 4 total | 4 available | 0 unavailable
    StrategyType:           RollingUpdate
    MinReadySeconds:        0
    RollingUpdateStrategy:  1 max unavailable, 1 max surge
    Pod Template:
      Labels:  app=helloworld
      Containers:
       k8s-demo:
        Image:      105552010/k8s-demo:v2
        Port:       3000/TCP
        Host Port:  0/TCP
        Limits:
          cpu:     200m
          memory:  100Mi
        Requests:
          cpu:        100m
          memory:     50Mi
        Environment:  <none>
        Mounts:       <none>
      Volumes:        <none>
    Conditions:
      Type           Status  Reason
      ----           ------  ------
      Available      True    MinimumReplicasAvailable
    OldReplicaSets:  <none>
    NewReplicaSet:   helloworld-deployment-9d595467 (4/4 replicas created)
    Events:
      Type    Reason             Age    From                   Message
      ----    ------             ----   ----                   -------
      Normal  ScalingReplicaSet  24m    deployment-controller  Scaled up replica set helloworld-deployment-5b5457694d to 2
      Normal  ScalingReplicaSet  11m    deployment-controller  Scaled up replica set helloworld-deployment-9d595467 to 1
      Normal  ScalingReplicaSet  11m    deployment-controller  Scaled down replica set helloworld-deployment-5b5457694d to 1
      Normal  ScalingReplicaSet  11m    deployment-controller  Scaled up replica set helloworld-deployment-9d595467 to 2
      Normal  ScalingReplicaSet  11m    deployment-controller  Scaled down replica set helloworld-deployment-5b5457694d to 0
      Normal  ScalingReplicaSet  6m14s  deployment-controller  Scaled up replica set helloworld-deployment-9d595467 to 3
      Normal  ScalingReplicaSet  3m38s  deployment-controller  Scaled up replica set helloworld-deployment-9d595467 to 4

### Rollback（倒回，降版）

將現有版本回朔到上一個版本。~~還記得瑪莎多拉嗎？~~

    $kubectl rollout undo deployment/helloworld-deployment
    deployment.extensions/helloworld-deployment rolled back

檢查倒回狀態

    $kubectl rollout status deployment/helloworld-deployment
    deployment "helloworld-deployment" successfully rolled out

檢查Image版本

    $kubectl describe deployment|grep -i image
        Image:      105552010/k8s-demo

檢查Pod狀態，應該沒變，只是改Image的版本而已

    $kubectl get po
    NAME                                     READY   STATUS    RESTARTS   AGE
    helloworld-deployment-5b5457694d-6p7pl   1/1     Running   0          75s
    helloworld-deployment-5b5457694d-ghhxw   1/1     Running   0          75s
    helloworld-deployment-5b5457694d-lfckb   1/1     Running   0          70s
    helloworld-deployment-5b5457694d-lgl7h   1/1     Running   0          71s

檢查rolling的紀錄

    $kubectl rollout history deployment/helloworld-deployment
    deployment.extensions/helloworld-deployment
    REVISION  CHANGE-CAUSE
    2         <none>
    3         <none>

最後恢復原狀

    $kubectl delete -f pod-deploy.yaml
    deployment.extensions "helloworld-deployment" deleted

Pod也會一併刪除

    $kubectl get po
    NAME                                     READY   STATUS        RESTARTS   AGE
    helloworld-deployment-5b5457694d-6p7pl   1/1     Terminating   0          5m43s
    helloworld-deployment-5b5457694d-ghhxw   1/1     Terminating   0          5m43s
    helloworld-deployment-5b5457694d-lfckb   1/1     Terminating   0          5m38s
    helloworld-deployment-5b5457694d-lgl7h   1/1     Terminating   0          5m39s
    $kubectl get po
    No resources found.

OK，基礎篇的AWS操作就到這邊囉～

# 小結

今天我們看到了在基礎篇整個運用在AWS上的樣子，基本上大同小異，~~只是會噴錢而已（笑~~，不過因為是遠端的連線，所以有時候會稍微卡一下，大致上使用還算是蠻順暢的，目前已經把之前基礎篇的部分大致瀏覽過，算是個完整的複習，當然我們也有看到新的部分，像是Scale up和Auto Scaling的部分，下一篇將會介紹進階篇的一些操作，我們明天見～

# 參考連結

- [Pod概觀](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)
- [Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)
- [Health checks](https://k8smeetup.github.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)
- [Deployment概述](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [ReplicaSet概述](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
