# [Day15] k8s管理篇（二）：Resource Quota、Namespaces

Author: Nick Zhuang
Type: kubernetes

# 前言

今天我們要來了解關於Resource Quota，也就是資源配額、資源分配的設定方式，在限制使用的設定上，我們可以利用這個元件去限制CPU、記憶體、GPU、磁碟空間等等。再來是Namespace的部分，命名空間，我們可以自行定義，並結合Resource Quota，藉以達到將集群的資源分群的效果，會先一併介紹兩個概念，並套用整合的例子。

# Resource Quota

資源配額，顧名思義是指在集群中能夠分配的資源，其實這段內容的一小部分在前面有稍微提過，主要是在介紹[Pod](https://ithelp.ithome.com.tw/articles/10219488)和[Deployment](https://ithelp.ithome.com.tw/articles/10219982)的時候。我們會在這邊進一步了解request和limit的詳細設定。

# Namespaces

命名空間可以視為在k8s中的虛擬集群，它們底層依賴於同一個物理集群，有幾個特性：

- 命名空間為名稱提供了一個範圍。資源的名稱需要在命名空間內是惟一的，但不能跨命名空間。命名空間不能嵌套在另外一個命名空間內，而且每個k8s資源只能屬於一個命名空間。
- 命名空間是在多個用戶之間劃分集群資源的一種方法（通過[資源配額](#Resource-Quota)）。
- 不需要使用多個命名空間來分隔輕微不同的資源，例如同一軟件的不同版本：使用Label（標籤）來區分同一命名空間中的不同資源。

# 整合範例

看完了前面兩個介紹，結合一起做一些例子方便理解：

新增一個YAML

    $vim resourcequota.yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: myspace
    ---
    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: compute-quota
      namespace: myspace
    spec:
      hard:
        requests.cpu: "1"
        requests.memory: 1Gi
        requests.nvidia.com/gpu: 1
        limits.cpu: "2"
        limits.memory: 2Gi
        limits.nvidia.com/gpu: 2
    ---
    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: object-quota
      namespace: myspace
    spec:
      hard:
        configmaps: "10"
        persistentvolumeclaims: "4"
        replicationcontrollers: "20"
        secrets: "10"
        services: "10"
        services.loadbalancers: "2"

新增Resource Quota

    $kubectl create -f resourcequota.yaml
    namespace/myspace created
    resourcequota/compute-quota created
    resourcequota/object-quota created

## 示例一：沒有Resource Quota的Deployment

編輯一個沒有設置Resource Quota的Deployment

    $vim helloworld-no-quotas.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: helloworld-deployment
      namespace: myspace
    spec:
      replicas: 3
      selector:
        matchLabels:
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
            - name: nodejs-port
              containerPort: 3000

接著執行它

    $kubectl apply -f helloworld-no-quotas.yaml
    deployment.apps/helloworld-deployment created

檢查集群狀態

    $kubectl get all -n myspace
    NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/helloworld-deployment   0/3     0            0           104s
    
    NAME                                               DESIRED   CURRENT   READY   AGE
    replicaset.apps/helloworld-deployment-649949477d   3         0         0       104s

顯然資源調度有問題，Deployment沒有完全開起來

用Describe詳細看一下ReplicaSet

    $kubectl describe replicaset.apps/helloworld-deployment-649949477d -n myspace
    Name:           helloworld-deployment-649949477d
    Namespace:      myspace
    Selector:       app=helloworld,pod-template-hash=649949477d
    Labels:         app=helloworld
                    pod-template-hash=649949477d
    Annotations:    deployment.kubernetes.io/desired-replicas: 3
                    deployment.kubernetes.io/max-replicas: 4
                    deployment.kubernetes.io/revision: 1
    Controlled By:  Deployment/helloworld-deployment
    Replicas:       0 current / 3 desired
    Pods Status:    0 Running / 0 Waiting / 0 Succeeded / 0 Failed
    Pod Template:
      Labels:  app=helloworld
               pod-template-hash=649949477d
      Containers:
       k8s-demo:
        Image:        105552010/k8s-demo
        Port:         3000/TCP
        Host Port:    0/TCP
        Environment:  <none>
        Mounts:       <none>
      Volumes:        <none>
    Conditions:
      Type             Status  Reason
      ----             ------  ------
      ReplicaFailure   True    FailedCreate
    Events:
      Type     Reason        Age                    From                   Message
      ----     ------        ----                   ----                   -------
      Warning  FailedCreate  8m38s                  replicaset-controller  Error creating: pods 
    "helloworld-deployment-649949477d-jg64g" is forbidden: failed quota: compute-quota: 
    must specify limits.cpu,limits.memory,requests.cpu,requests.memory

注意Event的地方，它是說沒有在Deployment裡面定義limit和request那些，所以不能用。

也就是說，當一個集群有分配ResourceQuota和對應的Namespace時，所有在該集群內使用元件時必須都要詳細指明用量，不然不給用！

## 示例二：有Resource Quota的Deployment

我們再試個例子，有ResourceQuota設置的Deployment

    $vim helloworld-with-quotas.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: helloworld-deployment
      namespace: myspace
    spec:
      replicas: 3
      selector:
        matchLabels:
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
            - name: nodejs-port
              containerPort: 3000
            resources:
              requests:
                cpu: 200m
                memory: 0.5Gi
              limits:
                cpu: 400m
                memory: 1Gi

注意到增加了`resources`這個區塊，其中的`request`是初始化跟系統要的資源，而`limit`則是最大允許用量。

啟動它

    $kubectl apply -f helloworld-with-quotas.yaml
    deployment.apps/helloworld-deployment created

檢查相關狀態

    $kubectl get all -n myspace
    NAME                                         READY   STATUS    RESTARTS   AGE
    pod/helloworld-deployment-5d87f4fbbf-rdsx7   1/1     Running   0          40s
    pod/helloworld-deployment-5d87f4fbbf-ttszb   1/1     Running   0          40s
    
    NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/helloworld-deployment   2/3     2            2           40s
    
    NAME                                               DESIRED   CURRENT   READY   AGE
    replicaset.apps/helloworld-deployment-5d87f4fbbf   3         2         2       40s

還少一個沒開起來，檢查下ResourceQuota

    $kubectl get resourcequotas -n myspace
    NAME            CREATED AT
    compute-quota   2019-09-29T16:32:49Z
    object-quota    2019-09-29T16:32:49Z
    $kubectl describe resourcequotas -n myspace
    Name:                    compute-quota
    Namespace:               myspace
    Resource                 Used    Hard
    --------                 ----    ----
    limits.cpu               600m    2
    limits.memory            1536Mi  2Gi
    limits.nvidia.com/gpu    0       2
    requests.cpu             300m    1
    requests.memory          768Mi   1Gi
    requests.nvidia.com/gpu  0       1
    
    
    Name:                   object-quota
    Namespace:              myspace
    Resource                Used  Hard
    --------                ----  ----
    configmaps              0     10
    persistentvolumeclaims  0     4
    replicationcontrollers  0     20
    secrets                 1     10
    services                0     10
    services.loadbalancers  0     2

接著檢查Pod狀態

    $kubectl describe replicasets.apps -n myspace
    Name:           helloworld-deployment-5d87f4fbbf
    Namespace:      myspace
    Selector:       app=helloworld,pod-template-hash=5d87f4fbbf
    Labels:         app=helloworld
                    pod-template-hash=5d87f4fbbf
    Annotations:    deployment.kubernetes.io/desired-replicas: 3
                    deployment.kubernetes.io/max-replicas: 4
                    deployment.kubernetes.io/revision: 1
    Controlled By:  Deployment/helloworld-deployment
    Replicas:       2 current / 3 desired
    Pods Status:    2 Running / 0 Waiting / 0 Succeeded / 0 Failed
    Pod Template:
      Labels:  app=helloworld
               pod-template-hash=5d87f4fbbf
      Containers:
       k8s-demo:
        Image:      105552010/k8s-demo
        Port:       3000/TCP
        Host Port:  0/TCP
        Limits:
          cpu:     400m
          memory:  1Gi
        Requests:
          cpu:        200m
          memory:     512Mi
        Environment:  <none>
        Mounts:       <none>
      Volumes:        <none>
    Conditions:
      Type             Status  Reason
      ----             ------  ------
      ReplicaFailure   True    FailedCreate
    Events:
      Type     Reason            Age                    From                   Message
      ----     ------            ----                   ----                   -------
      Normal   SuccessfulCreate  3m35s                  replicaset-controller  Created pod: helloworld-deployment-5d87f4fbbf-ttszb
      Warning  FailedCreate      3m35s                  replicaset-controller  Error creating: pods "helloworld-deployment-5d87f4fbbf-h7krq" is forbidden: exceeded quota: 
    compute-quota, requested: limits.memory=1Gi,requests.memory=512Mi, used: limits.memory=2Gi,requests.memory=1Gi, limited: limits.memory=2Gi,requests.memory=1Gi

顯然資源不夠分，因為ReplicaSet是3，如果3個開滿會吃到3G的記憶體，但是現在限制是2G

刪掉這個Deployment

    $kubectl delete -f helloworld-with-quotas.yaml
    deployment.apps "helloworld-deployment" deleted

## 示例三：Limit Range搭配沒有Resource Quota的Deployment

新增一個Limit Range

    $vim defaults.yaml
    apiVersion: v1
    kind: LimitRange
    metadata:
      name: limits
      namespace: myspace
    spec:
      limits:
      - default:
          cpu: 200m
          memory: 512Mi
        defaultRequest:
          cpu: 100m
          memory: 256Mi
        type: Container

啟動它

    $kubectl create -f defaults.yaml
    limitrange/limits created

檢查Limit Range狀態

    $kubectl describe limitranges -n myspace
    Name:       limits
    Namespace:  myspace
    Type        Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
    ----        --------  ---  ---  ---------------  -------------  -----------------------
    Container   cpu       -    -    100m             200m           -
    Container   memory    -    -    256Mi            512Mi          -

啟動沒有Resource Quota的Deployment

    $kubectl create -f helloworld-no-quotas.yaml
    deployment.apps/helloworld-deployment created

檢查myspace的集群狀態

    $kubectl get all -n myspace
    NAME                                         READY   STATUS    RESTARTS   AGE
    pod/helloworld-deployment-649949477d-bxwf7   1/1     Running   0          81s
    pod/helloworld-deployment-649949477d-h7rlb   1/1     Running   0          81s
    pod/helloworld-deployment-649949477d-qznl2   1/1     Running   0          81s
    
    NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/helloworld-deployment   3/3     3            3           81s
    
    NAME                                               DESIRED   CURRENT   READY   AGE
    replicaset.apps/helloworld-deployment-649949477d   3         3         3       81s

如果Namespace沒有要用的話，可以

    $kubectl delete namespaces myspace
    namespace "myspace" deleted

這樣就會把myspace底下所有相關的資源一併刪除。

OK，測試成功！

# 小結

今天我們學習到了Resource Quota，發現到除了在Pod和Deployment內可以定義外，這是一個全域定義的設置，只要在集群內沒有定義這個的就不能啟動，達到整合集群的管理資源效果。我們也看到了Namespace，一個命名空間，實際上的意義是虛擬群集，它有效的將原先集群內部的範圍做了很好的劃分，最後我們看了幾個整合應用，知道了Resource Quota可以搭配Namespace來使用，這樣的做法可以有效的區分不同虛擬群集的資源分配工作。

# 參考資料：

- [Resource Quota簡介](https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/)
- [Namespace簡介](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Limit Range](https://kubernetes.io/docs/tasks/administer-cluster/limit-storage-consumption/)
- [Schedule GPU](https://kubernetes.io/zh/docs/tasks/manage-gpus/scheduling-gpus/)
