# 第十三天：k8s之六：進階篇之四，Affinity and Anti-Affinity、Taints and Tolerations

Author: Nick Zhuang
Type: kubernetes

# 前言

今天我們要來談談一些管理k8s群集的時候，有可能會用到的設定，這部分我把它放在進階篇最後，主要是因為它算是一種過渡，這兩個topic，橫跨了進階篇和管理篇的內容，首先我們看到affinity和Anti-Affinity，所謂的Affinity是親和性的意思，在k8s中講到Affinity就是在指親和性調度，意思就是當滿足了某些條件，就把Pod調度到集群上，Anti-Affinity則是相反的邏輯，也就是當也就是當滿足了某些條件，就不要把Pod調度到集群上。再來我們要看的是Taint（污點）和Toleration（容忍），Taint提供了Node一項標記，但注意，這個標記不是Label，當設置好Taint後，滿足Toleration的Pod將可以部署到該Node上。

# Affinity and Anti-Affinity

Affinity就是親和性，這個部分分為Node Affinity和Pod Affinity，下面給幾個例子：

## Node Selector

還記得前面提過的Label嗎？我們其實可以透過設置nodeSelector來達成Node Affinity的需求

按照慣例，我們從helloworld的Pod開始～

新增一個具有nodeSelector的Pod

    $vim pod-nodeSelector.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nodehelloworld.example.com
      labels:
        app: helloworld
    spec:
      containers:
      - name: k8s-demo
        image: 105552010/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000
      nodeSelector:
        disk: ssd

檢查集群的Node狀態，包含Labels

    $kubectl get no --show-labels
    NAME       STATUS   ROLES    AGE   VERSION   LABELS
    minikube   Ready    master   46h   v1.16.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=minikube,kubernetes.io/os=linux,node-role.kubernetes.io/master=

我們來新增這個Pod

    $kubectl create -f pod-nodeSelector.yaml
    pod/nodehelloworld.example.com created

現在因為Node沒有設置disk=ssd這個Label，就會長這樣（Pod會一直Pending

    $kubectl get po
    NAME                         READY   STATUS    RESTARTS   AGE
    nodehelloworld.example.com   0/1     Pending   0          4s

在describe中的Event可以發現原因：不滿足nodeSelector的條件

    $kubectl describe po
    Events:
      Type     Reason            Age        From               Message
      ----     ------            ----       ----               -------
      Warning  FailedScheduling  <unknown>  default-scheduler  0/1 nodes are available: 1 node(s) didn't match node selector.

這時我們新增一個disk=ssd的Label並檢查

    $kubectl label nodes minikube disk=ssd
    node/minikube labeled
    $kubectl get no --show-labels
    NAME       STATUS   ROLES    AGE   VERSION   LABELS
    minikube   Ready    master   46h   v1.16.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disk=ssd,kubernetes.io/arch=amd64,kubernetes.io/hostname=minikube,kubernetes.io/os=linux,node-role.kubernetes.io/master=

可以發現是有新增disk=ssd這個Label！

我們再來檢查下Pod的狀態

    $kubectl get po
    NAME                         READY   STATUS    RESTARTS   AGE
    nodehelloworld.example.com   1/1     Running   0          6m53s

有開始跑了，如果再度檢查describe的訊息中的Event會發現：

    Successfully assigned default/nodehelloworld.example.com to minikube

附帶一提，如果Node要解除Label的話，可以這樣

    $kubectl label nodes minikube disk-
    node/minikube labeled
    $kubectl get no --show-labels
    NAME       STATUS   ROLES    AGE    VERSION   LABELS
    minikube   Ready    master   2d6h   v1.16.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=minikube,kubernetes.io/os=linux,node-role.kubernetes.io/master=

OK，接著我們看下個部分，Node Affinity

## Node Affinity

這個部分我們就要把前一段的Node Selector抽換掉，把它改成Node Affinity，Node Affinity和Node Selector不同的是，它會有更多細緻的操作，你可以把Node Selector看成是簡易版的Node Affinity，k8s最早只有Node Selector，Affinity是後面的版本才加進來的。

nodeAffinity有兩種策略：preferredDuringSchedulingIgnoredDuringExecution和requiredDuringSchedulingIgnoredDuringExecution，前面的就是軟策略（可以不滿足，不會怎樣），後面的就是硬策略（一定要滿足，不然就翻臉）。

新增一個Node Affinity的Pod

    $vim pod-nodeAffinity.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nodehelloworld.example.com
      labels:
        app: helloworld
    spec:
      containers:
      - name: k8s-demo
        image: 105552010/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: NotIn
                values:
                - 192.168.0.125
                - 192.168.0.247
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: disk
                operator: In
                values:
                - ssd

可以看到在affinity的區塊，有設置了nodeAffinity，其中requiredDuringSchedulingIgnoredDuringExecution代表了硬需求，注意到這裡寫的nodeSelectorTerms，在這個區塊底下，如果沒有寫matchExpressions的話，那麼在它底下所列出的條件，只要滿足其中一項就可以了，但在這個例子有寫matchExpressions，所以在matchExpressions底下所列的條件全都要滿足才會調度，關於matchExpressions的key有些系統本身自帶的可以選擇自行套用，可以參考下：

    |build-in node label                      |
    |-----------------------------------------|
    |kubernetes.io/hostname                   |
    |failure-domain.beta.kubernetes.io/zone   |
    |failure-domain.beta.kubernetes.io/region |
    |beta.kubernetes.io/instance-type         |
    |beta.kubernetes.io/os                    |
    |beta.kubernetes.io/arch                  |

這邊是檢查主機名稱：kubernetes.io/hostname

接著我們看到了operator，這個是操作符，一般會有這幾種：

- In：Label 的值在某個列表中
- NotIn：Label 的值不在某個列表中
- Gt：Label 的值大於某個值
- Lt：Label 的值小於某個值
- Exists：某個Label 存在
- DoesNotExist：某個Label 不存在

這裡是用NotIn，也就是說，required那邊整體的邏輯是：強制規定不把Pod部署到192.168.0.125和192.168.0.247這兩台電腦上。

再來是preferredDuringSchedulingIgnoredDuringExecution的部分，軟性要求，可以看到這裡有個weight，那是權重的意思，1~100，越高代表越優先考慮這個條件，最後是matchExpressions，這邊規定了要調度Pod到node label 為disk=ssd的Node上。

我們來測試下：

    $kubectl apply -f pod-nodeAffinity.yaml
    pod/nodehelloworld.example.com created

檢查Pod狀態

    $kubectl get po
    NAME                         READY   STATUS              RESTARTS   AGE
    nodehelloworld.example.com   0/1     ContainerCreating   0          4s
    $kubectl get po
    NAME                         READY   STATUS    RESTARTS   AGE
    nodehelloworld.example.com   1/1     Running   0          8s

OK，看起來沒啥問題。

介紹完Node Affinity的內容後，我們來看下Pod Affinity的部分

## Pod Affinity

和nodeAffinity類似，podAffinity也有requiredDuringSchedulingIgnoredDuringExecution和preferredDuringSchedulingIgnoredDuringExecution兩種調度策略，唯一不同的是如果要使用互斥性，我們需要使用podAntiAffinity字段。

這邊我們新增一個YAML

    $vim pod-podAffinity.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nodehelloworld.example.com
      labels:
        app: helloworld
    spec:
      containers:
      - name: k8s-demo
        image: 105552010/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - helloworld
            topologyKey: kubernetes.io/hostname
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - node-affinity-pod
              topologyKey: kubernetes.io/hostname

相關參數說明請參考Node Affinity講解的部分，這裡不再贅述。

總結來看，這個設定是要在同個名稱的Node底下，要部署Label中app=helloworld的Pod，而不要部署Label中app=node-affinity-pod的Pod

我們來新增這個Pod試試

    $kubectl create -f pod-podAffinify.yaml
    pod/nodehelloworld.example.com created

檢查Pod的狀態

    $kubectl get po
    NAME                         READY   STATUS              RESTARTS   AGE
    nodehelloworld.example.com   0/1     ContainerCreating   0          4s
    $kubectl get po
    NAME                         READY   STATUS    RESTARTS   AGE
    nodehelloworld.example.com   1/1     Running   0          6s

看起來運行正常，補充說明下：

> 在labelSelector和topologyKey的同級，還可以定義namespaces列表，表示匹配哪些namespace裡面的pod，默認情況下，會匹配定義的pod所在的namespace；如果定義了這個字段，但是它的值為空，則匹配所有的namespaces。

P.S. namespace的觀念，後面管理篇會介紹

最後我們整理一下比較表

    |調度策略         |匹配標籤 |操作符                                   |拓撲域支持   |調度目標              |
    |----------------|-------|----------------------------------------|----------- |--------------------|
    |nodeAffinity    |Node   |In, NotIn, Exists, DoesNotExist, Gt, Lt |否          |指定主機              |
    |podAffinity     |Pod    |In, NotIn, Exists, DoesNotExist         |是          |Pod與指定Pod同一拓撲域 |
    |podAnitAffinity |Pod    |In, NotIn, Exists, DoesNotExist         |是          |Pod與指定Pid同一拓撲域 |

好的，所有Affinity的部分到這邊結束。

# Taints and Tolerations

污點（Taints）與容忍（tolerations），對於nodeAffinity無論是硬策略還是軟策略方式，都是調度Pod到預期節點上，而Taints恰好與之相反，如果一個節點標記為Taints ，除非Pod也被標識為可以容忍污點節點，否則該Taints節點不會被調度Pod。

我們來看幾個例子：

## 示例一：強制踢出

先標記一個Node，當然因為我們現在是單節點群集，所以只能標記minikube這個節點

    $kubectl taint nodes minikube app=helloworld:NoExecute
    node/minikube tainted

Taint的策略有三種：

1. `NoSchedule`：POD 不會被調度到標記為taints 節點。
2. `PreferNoSchedule`：NoSchedule 的軟策略版本。
3. `NoExecute`：該選項意味著一旦Taint 生效，如該節點內正在運行的POD 沒有對應Tolerate 設置，會直接被逐出。

檢查設置的Taint

    $kubectl describe nodes|grep -i taint
    Taints:             app=helloworld:NoExecute

新增一個Pod的YAML

    $vim pod-podAffinity-taint.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nodehelloworld.example.com
      labels:
        app: helloworld
    spec:
      containers:
      - name: k8s-demo
        image: 105552010/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - helloworld
            topologyKey: kubernetes.io/hostname
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - node-affinity-pod
              topologyKey: kubernetes.io/hostname
      tolerations:
      - key: "app"
        operator: "Equal"
        value: "helloworld"
        effect: "NoExecute"
        tolerationSeconds: 60

跑一個Pod Affinity的Pod，當Taint生效的時候，Pod就會被強制關閉，這邊有設定緩衝時間60秒。~~就像拍電影一樣：驚天動地60秒~~

    $kubectl get po
    NAME                         READY   STATUS        RESTARTS   AGE
    nodehelloworld.example.com   1/1     Terminating   0          50m
    $kubectl get po
    NAME                         READY   STATUS        RESTARTS   AGE
    nodehelloworld.example.com   0/1     Terminating   0          51m
    $kubectl get po
    No resources found.

當不需要標記的時候，我們可以把Taint拿掉

    $kubectl taint nodes minikube app-
    node/minikube untainted

## 示例二：限制排程

按照剛所說的Taint策略，我們再試試另一種：NoSchedule

    $kubectl taint nodes minikube app=helloworld:NoSchedule
    node/minikube tainted

接著我們修改下剛剛pod-podAffinity-taint.yaml的內容

    $vim pod-podAffinity-taint.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nodehelloworld.example.com
      labels:
        app: helloworld
    spec:
      containers:
      - name: k8s-demo
        image: 105552010/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - helloworld
            topologyKey: kubernetes.io/hostname
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - node-affinity-pod
              topologyKey: kubernetes.io/hostname
      tolerations:
      - key: "app"
        operator: "Equal"
        value: "helloworld"
        effect: "NoSchedule"

啟動這個Pod

    $kubectl create -f pod-podAffinify-taint.yaml
    pod/nodehelloworld.example.com created

檢查Pod的狀態，發現是Pending

    $kubectl get po
    NAME                         READY   STATUS    RESTARTS   AGE
    nodehelloworld.example.com   0/1     Pending   0          4s

這是因為沒有Node有app=helloworld這個Taint，我們把這個Taint加到minikube的Node上

    $kubectl taint node minikube app=helloworld:NoSchedule
    node/minikube tainted

檢查Node的Taint

    $kubectl describe no minikube|grep -i taint
    Taints:             app=helloworld:NoSchedule

再次確認Pod的狀態

    $kubectl get po
    NAME                         READY   STATUS              RESTARTS   AGE
    nodehelloworld.example.com   0/1     ContainerCreating   0          4s
    $kubectl get po
    NAME                         READY   STATUS    RESTARTS   AGE
    nodehelloworld.example.com   1/1     Running   0          6s

有開起來了！測試成功～

# 小結

今天我們看到了Affinity與Anti-Affinity（親和性與反親和性），以及Taints和Tolerations（污點和容忍），這幾個都是在限制或是放寬部署條件的時候很好用的設置，透過這些元件的不同組合，我們可以達到不同場景下自動部署的條件。明天會開始介紹k8s管理篇的部分，敬請期待！

# 參考連結

- [Taint與Toleration](https://kubernetes.io/zh/docs/concepts/configuration/taint-and-toleration/)
- [Assign Pod to Node](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)

