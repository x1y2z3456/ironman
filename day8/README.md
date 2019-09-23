# 第八天：k8s之一：基礎篇之一，Pod、Label、Health check、Scaling

Author: Nick Zhuang
Type: kubernetes

# 前言

前幾天我們已經大致理解k8s上的架構，我們也從幾個不同的面向去理解k8s，那麼從今天開始，我們就來仔細看看k8s架構中的所有細節，基礎篇的部分，我將它切成兩塊，透過引導對話的方式去介紹，這樣會比較好理解。那我們就開始吧！

# 前置作業

先確定有把minikube開起來（mac的啟動方式比較炫砲

    $minikube start
    😄  minikube v0.35.0 on darwin (amd64)
    💡  Tip: Use 'minikube start -p <name>' to create a new cluster, or 'minikube delete' to delete this one.
    🔄  Restarting existing virtualbox VM for "minikube" ...
    ⌛  Waiting for SSH access ...
    📶  "minikube" IP address is 192.168.99.100
    🐳  Configuring Docker as the container runtime ...
    ✨  Preparing Kubernetes environment ...
    🚜  Pulling images required by Kubernetes v1.13.4 ...
    🔄  Relaunching Kubernetes v1.13.4 using kubeadm ...
    ⌛  Waiting for pods: apiserver proxy etcd scheduler controller addon-manager dns
    📯  Updating kube-proxy configuration ...
    🤔  Verifying component health ......
    💗  kubectl is now configured to use "minikube"
    🏄  Done! Thank you for using minikube!

這樣就沒問題了，接著往下一步

# Pod（豆莢）

這部分之前有簡略談過，我們來複習下：

Pod是k8s中可以被創造或是發布的最小的單位，一個 Pod 在k8s世界中就相當於一個application。Pod有以下特點：
1. 每個 Pod 都有屬於自己的 yaml 檔
2. 一個 Pod 裡面可以包含一個或多個 Docker Container
3. 在同一個 Pod 裡面的 containers，可以用 local port 來互相溝通

我們先新增一個pod文件

    $vim pod-demo.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-helloworld
    spec:
      containers:
      - name: k8s-demo
        image: 105552010/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000

我們開啟一個pod

    $kubectl apply -f pod-demo.yaml
    pod/my-helloworld created

接著我們來看看集群裡的pod

    $kubectl get po
    NAME            READY   STATUS    RESTARTS   AGE
    my-helloworld   1/1     Running   0          2m21s

Pod的狀態
|Value    |Description   |
|---------|--------------|
|Pending  |Pod已被k8s接受，但尚未創建完成，這可能需要一段時間。|
|Running  |Pod已創建所有Container。至少有一個Container仍在運行，或者正在啟動或重新啟動。|
|Succeeded|Pod中的所有Container都已成功終止，並且不會重新啟動。|
|Failed   |Pod中的所有Container都已終止，Container不是退出非零狀態，就是被系統終止。|
|Unknown  |由於某種原因，無法獲得Pod的狀態，這通常是由於與Pod的主機通信時出錯。|

看起來沒啥問題，不過讓我們思考一個問題：

現在的情況是只有一個pod在跑，當有很多個pod在跑的時候，我們要如何能很快速的找到我們的pod？

答案顯而易見，就是用pod的NAME去找，不過當我們需要找到一群類似性質的pod時，我們可以怎麼做呢？

這個部分就牽涉到下個內容了：Label。

# Label（標籤）

我們可以為我們的pod定義一個標籤，舉例來說：

修改pod-demo.yaml的內容（紅字內容）

    $vim pod-demo.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-helloworld
      **labels:
        app: helloworld**
    spec:
      containers:
      - name: k8s-demo
        image: 105552010/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000

注意到這個Label所對應的Key=Value值對應為app=helloworld

先把原來的pod刪掉

    $kubectl delete po my-helloworld
    pod "my-helloworld" deleted

開啟新的pod

    $kubectl apply -f pod-demo.yaml
    pod/my-helloworld created

檢查pod狀態（顯示標籤模式

    $kubectl get po --show-labels
    NAME            READY   STATUS    RESTARTS   AGE   LABELS
    my-helloworld   1/1     Running   0          67s   app=helloworld

到這邊應該沒啥問題（k8s中還有個類似Label的東東，那就是[Annotation](https://jimmysong.io/kubernetes-handbook/concepts/annotation.html)），不過我們可以思考下，是否能自動去檢查應用（app）的狀態呢？因為在有些情況下，pod和container是正常的，但是應用（app）卻不能使用，這時候我們就會需要有個機制去幫我們把pod重啟，也就是下個章節：Health checks。

# Health checks（健康檢查）

這個章節分為兩個部分：

- liveness probe：針對pod的自我健康狀態檢查，若有異常pod會重啟container
- readiness probe：針對pod的可access的檢查，但和liveness probe不同的是，如果檢查有異常，他不會重啟container，而是將它的IP設定移除，這樣一來它就不能被任何地方連線

我們先來看liveness probe的例子：

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
        ports:
        - name: nodejs-port
          containerPort: 3000
        **livenessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 15
          timeoutSeconds: 30**

這裡說明下這些參數的使用：

httpGet是用來檢查執行http request的時候是否正常，其中path是指訪問的HTTP服務器的路徑、port是指訪問的容器的端口名字或者端口號（必須介於1和65525之間）。

initialDelaySeconds：容器啟動後第一次執行探測是需要等待多少秒，這邊是15秒。

timeoutSeconds：探測超時時間默認30秒，最小1秒。

我們來測試下：

    $kubectl delete po my-helloworld
    pod "my-helloworld" deleted
    $kubectl apply -f pod-demo.yaml
    pod/my-helloworld created
    $kubectl get po --show-labels
    NAME            READY   STATUS    RESTARTS   AGE   LABELS
    my-helloworld   1/1     Running   0          17s   app=helloworld
    $kubectl describe po my-helloworld
    #僅列出Containers的部分
    Containers:
      k8s-demo:
        Container ID:   docker://fc9782038190720ffdbfa5d8c12544b9032c4bf658b24dd7593fb38a589ddbd7
        Image:          105552010/k8s-demo
        Image ID:       docker-pullable://105552010/k8s-demo@sha256:2c050f462f5d0b3a6430e7869bcdfe6ac48a447a89da79a56d0ef61460c7ab9e
        Port:           3000/TCP
        Host Port:      0/TCP
        State:          Running
          Started:      Sat, 14 Sep 2019 23:03:48 +0800
        Ready:          True
        Restart Count:  0
        Liveness:       http-get http://:3000/ delay=15s timeout=30s period=10s #success=1 #failure=3
        Environment:    <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from default-token-n565q (ro)

可以看到liveness的設定！

接著我們在加入readiness probe的設定（是否ready的檢查條件）：

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
        **- /bin/sh
        - -c
        - touch /tmp/healthy; npm start; sleep 30; rm -rf /tmp/healthy; sleep 600**
        ports:
        - name: nodejs-port
          containerPort: 3000
        livenessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 15
          timeoutSeconds: 30
        **readinessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 5
          periodSeconds: 5**

這邊的意思是說，container啟動的時候會先執行touch那行指令

而readiness會去檢查它產生的檔案，也就是/tmp/healthy，若沒問題pod就會變成Ready的狀態

initialDelaySeconds：容器啟動後第一次執行探測是需要等待多少秒，這邊是5秒。

periodSeconds：執行探測的頻率默認是10秒，最小1秒。這邊設置5秒。

    $kubectl delete po my-helloworld
    pod "my-helloworld" deleted
    $kubectl create -f pod-demo.yaml
    pod/my-helloworld created
    $kubectl get po
    NAME            READY   STATUS    RESTARTS   AGE
    my-helloworld   1/1     Running   0          111s
    $kubectl describe po
    Name:               my-helloworld
    Namespace:          default
    Priority:           0
    PriorityClassName:  <none>
    Node:               minikube/10.0.2.15
    Start Time:         Sun, 15 Sep 2019 02:05:59 +0800
    Labels:             app=helloworld
    Annotations:        <none>
    Status:             Running
    IP:                 172.17.0.2
    Containers:
      k8s-demo:
        Container ID:  docker://0795cfffce439b787d663b0ce6ca911e7726a0bcb198b356349a75d1c73085b6
        Image:         105552010/k8s-demo
        Image ID:      docker-pullable://105552010/k8s-demo@sha256:2c050f462f5d0b3a6430e7869bcdfe6ac48a447a89da79a56d0ef61460c7ab9e
        Port:          3000/TCP
        Host Port:     0/TCP
        Args:
          /bin/sh
          -c
          touch /tmp/healthy; npm start; sleep 60; rm -rf /tmp/healthy; sleep 600
        State:          Running
          Started:      Sun, 15 Sep 2019 02:06:06 +0800
        Ready:          True
        Restart Count:  0
        Liveness:       http-get http://:3000/ delay=15s timeout=30s period=10s #success=1 #failure=3
        Readiness:      exec [cat /tmp/healthy] delay=5s timeout=1s period=5s #success=1 #failure=3
        Environment:    <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from default-token-n565q (ro)
    Conditions:
      Type              Status
      Initialized       True
      Ready             True
      ContainersReady   True
      PodScheduled      True
    Volumes:
      default-token-n565q:
        Type:        Secret (a volume populated by a Secret)
        SecretName:  default-token-n565q
        Optional:    false
    QoS Class:       BestEffort
    Node-Selectors:  <none>
    Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                     node.kubernetes.io/unreachable:NoExecute for 300s
    Events:
      Type    Reason     Age   From               Message
      ----    ------     ----  ----               -------
      Normal  Scheduled  45s   default-scheduler  Successfully assigned default/my-helloworld to minikube
      Normal  Pulling    44s   kubelet, minikube  pulling image "105552010/k8s-demo"
      Normal  Pulled     39s   kubelet, minikube  Successfully pulled image "105552010/k8s-demo"
      Normal  Created    39s   kubelet, minikube  Created container
      Normal  Started    38s   kubelet, minikube  Started container

好的，這樣看起就很完美了，但是，如果我們希望可以一次開很多個pod以防萬一，甚至是每個pod都能去指定它所能夠調配資源呢？像是CPU或RAM之類的，有可能做到嗎？

有的！這就是Scaling的部分，也就是下個章節。

# Scaling（拓展）

一般來說，Scaling分為兩種：

- Vertical Scaling：水平拓展，係指能夠分配更多的系統資源給Pod
- Horizontal Scaling：垂直拓展，係指可以創建分身，但是本身必須是Stateless的Pod，Stateless是指沒有在本地端寫入資料

pod的Horizontal Scaling必須要用到Replication Controller這個元件：

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

創建Replication Controller

    $kubectl create -f pod-scale.yaml
    replicationcontroller/helloworld-controller created

檢查Replication Controller狀態

    $kubectl get rc
    NAME                    DESIRED   CURRENT   READY   AGE
    helloworld-controller   2         2         1       67s
    $kubectl describe rc helloworld-controller
    Name:         helloworld-controller
    Namespace:    default
    Selector:     app=helloworld
    Labels:       app=helloworld
    Annotations:  <none>
    Replicas:     2 current / 2 desired
    Pods Status:  2 Running / 0 Waiting / 0 Succeeded / 0 Failed
    Pod Template:
      Labels:  app=helloworld
      Containers:
       k8s-demo:
        Image:        105552010/k8s-demo
        Port:         3000/TCP
        Host Port:    0/TCP
        Environment:  <none>
        Mounts:       <none>
      Volumes:        <none>
    Events:
      Type    Reason            Age   From                    Message
      ----    ------            ----  ----                    -------
      Normal  SuccessfulCreate  86s   replication-controller  Created pod: helloworld-controller-dwvng

可以看到它已經幫我們開了共2個pod

好的，到這邊，再來就是，我們為這個YAML增加Vertical Scaling的設定

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

這邊的CPU設置是按照電腦上的核心去決定的1個核心代表1000m，最小配置以100m為單位

以及記憶體的配置是50MB，如果要配10GB就輸入10Gi即可

然後我們更新一下這個Replication Controller，並檢查它的狀態

    $kubectl apply -f pod-scale.yaml
    Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
    replicationcontroller/helloworld-controller configured
    $kubectl describe rc helloworld-controller
    Name:         helloworld-controller
    Namespace:    default
    Selector:     app=helloworld
    Labels:       app=helloworld
    Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                    {"apiVersion":"v1","kind":"ReplicationController","metadata":{"annotations":{},"name":"helloworld-controller","namespace":"default"},"spec...
    Replicas:     2 current / 2 desired
    Pods Status:  2 Running / 0 Waiting / 0 Succeeded / 0 Failed
    Pod Template:
      Labels:  app=helloworld
      Containers:
       k8s-demo:
        Image:       105552010/k8s-demo
        Ports:       300/TCP, 3000/TCP
        Host Ports:  0/TCP, 0/TCP
        Requests:
          cpu:        100m
          memory:     50Mi
        Environment:  <none>
        Mounts:       <none>
      Volumes:        <none>
    Events:
      Type    Reason            Age   From                    Message
      ----    ------            ----  ----                    -------
      Normal  SuccessfulCreate  19m   replication-controller  Created pod: helloworld-controller-dwvng

可以看到多了Request那個部分

接著我們來看看集群pod狀態

    $kubectl get po
    NAME                          READY   STATUS    RESTARTS   AGE
    helloworld-controller-4vbgm   1/1     Running   0          34s
    helloworld-controller-hbdp8   1/1     Running   0          34s

有兩個pod在跑了，大功告成！

# 小結

我們今天學習了基礎篇的一些知識，這些內容提到在k8s中最小的單位是Pod，它是構成整個系統最基礎的結構，一個Pod可以開多個container，我們也可以用Label去標記它，這個Label是可以有多個的，此外，我們還可以幫它設定自動檢查（Health checks），檢查它執行是否活著（liveness probe）、以及是否為Ready（readiness probe）等等狀態，最後我們透過Scaling，讓它可以透過水平拓展（Vertical Scaling）製作分身、以及垂直拓展（Horizontal Scaling）去分配系統資源。

明天會接續基礎篇的介紹，我們明天見！

# 參考資料

- [Pod概觀](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)
- [Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)
- [Health checks](https://k8smeetup.github.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)
