# [Day12] k8s進階篇（三）：StatefulSet、DaemonSet

Author: Nick Zhuang
Type: kubernetes

# 前言

今天我們要來聊聊關於一些排程的東西，簡單來說，所謂的StatefulSet就是一組有編號的Pod集合，每一個Pod都會有對應的獨立編號。再來是DaemonSet，這也是關於Pod的進階管控元件，它可以維持Pod運行的狀態，提供一個自動重啟的保證。

# 寫在前面

這裡稍微講下實務上常碰到的問題，就是當我們看到網上有個不錯的YAML可以用，很開心的把它複製下來，但卻不能用，因為apiVersion不支援，如果是用kubectl api-versions列出來一個個查找實在超花時間，改到天荒地老都弄不好，這時候我們可以怎麼辦呢？

有的，使用對應元件的spec去查找：

    $kubectl explain deployment.spec |grep -i version
    VERSION:  apps/v1

所以我們知道，在現在的這個環境，Deployment的YAML中的apiVersion要設為apps/v1

如果沒用grep的話，它會列出詳細元件的使用方式，類似於我們在用man查詢linux指令的效果

再介紹另外一個小技巧：

我們在用kubectl下指令的時候，可以利用一些元件的簡寫去簡化操作（參考Cheat Sheet），不過有更簡單的做法，就是設置AutoComplete，可以參考：[kubectl命令補全](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-autocomplete)，這個設置了之後，操作kubectl + TAB，它會給你提示，可能的話會自動幫你把命令補全，不錯用。

後續會把一些小技巧整理成一篇，順便做複習，有架構也比較好吸收，前面不提的原因是，這剛開始就設置了就不會感受到它的強大效果。

# StatefulSet

這個部分前面有稍微提過，我們再複習下：

StatefulSet管理一組Pod的部署（Deployment）和擴展（scaling），並提供有關這些Pod的排序和唯一性的保證。
與部署（Deployment）類似的是，StatefulSet管理基於相同容器規範的Pod。與部署（Deployment）不同的是，StatefulSet為其Pod保持標籤（label）。
這些Pod是根據相同的規範創建的，但不可互換：每個Pod都有一個永久性的標籤（label），它可以在任何重新安排時保留。

Stateful是指有狀態的、Stateless是指無狀態，也就是說當應用（app）需要狀態的時候，就會要用StatefulSet來設置。

我們來看個例子：

    $vim pod-statefulSet.yaml
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: helloworld-statefulset
    spec:
      selector:
        matchLabels:
          app: helloworld
      serviceName: "helloworld"
      replicas: 3
      template:
        metadata:
          labels:
            app: helloworld
        spec:
          containers:
          - name: k8s-demo
            image: 105552010/k8s-demo:v1
            ports:
            - containerPort: 3000

接著我們建立一個StatefulSet

    $kubectl create -f pod-statefulSet.yaml
    statefulset.apps/helloworld-statefulset created

檢查詳細狀態

    $kubectl get statefulsets.apps
    NAME                     READY   AGE
    helloworld-statefulset   3/3     4m44s
    $kubectl get po
    NAME                       READY   STATUS    RESTARTS   AGE
    helloworld-statefulset-0   1/1     Running   0          7m14s
    helloworld-statefulset-1   1/1     Running   0          7m54s
    helloworld-statefulset-2   1/1     Running   0          8m44s

可以注意到Pod有編號！在更新的時候，它會按照順序更新

我們來測試幾個操作：

## Scaling（拓展）

Scale up，擴充增加副本

    $kubectl scale statefulset --replicas 5 helloworld-statefulset
    statefulset.apps/helloworld-statefulset scaled

可以注意到Pod從3個變5個了

    $kubectl get po
    NAME                       READY   STATUS    RESTARTS   AGE
    helloworld-statefulset-0   1/1     Running   0          25m
    helloworld-statefulset-1   1/1     Running   0          25m
    helloworld-statefulset-2   1/1     Running   0          25m
    helloworld-statefulset-3   1/1     Running   0          5s
    helloworld-statefulset-4   1/1     Running   0          3s

Scale down，縮小減少副本

    $kubectl get po
    NAME                       READY   STATUS    RESTARTS   AGE
    helloworld-statefulset-0   1/1     Running   0          38m
    helloworld-statefulset-1   1/1     Running   0          38m
    helloworld-statefulset-2   1/1     Running   0          38m

## Update（更新，升版）

更新pod所用的image版本（將v1更新至v2

    $kubectl set image statefulset/helloworld-statefulset k8s-demo=105552010/k8s-demo:v2
    statefulset.apps/helloworld-statefulset image updated

檢查Pod，它會依序更新

    $kubectl get po
    NAME                       READY   STATUS        RESTARTS   AGE
    helloworld-statefulset-0   1/1     Running       0          43m
    helloworld-statefulset-1   1/1     Running       0          43m
    helloworld-statefulset-2   1/1     Terminating   0          43m
    $kubectl get po
    NAME                       READY   STATUS        RESTARTS   AGE
    helloworld-statefulset-0   1/1     Running       0          44m
    helloworld-statefulset-1   1/1     Terminating   0          44m
    helloworld-statefulset-2   1/1     Running       0          26s
    $kubectl get po
    NAME                       READY   STATUS        RESTARTS   AGE
    helloworld-statefulset-0   1/1     Terminating   0          44m
    helloworld-statefulset-1   1/1     Running       0          5s
    helloworld-statefulset-2   1/1     Running       0          55s
    $kubectl get po
    NAME                       READY   STATUS    RESTARTS   AGE
    helloworld-statefulset-0   1/1     Running   0          64s
    helloworld-statefulset-1   1/1     Running   0          104s
    helloworld-statefulset-2   1/1     Running   0          2m34s

可以比較一下之前Deployment的操作，值得一提的是，StatefulSet並不支援Pause和Resume的操作喔

    $kubectl rollout pause statefulsets.apps helloworld-statefulset
    error: statefulsets.apps "helloworld-statefulset" pausing is not supported
    $kubectl rollout resume statefulsets.apps helloworld-statefulset
    error: statefulsets.apps "helloworld-statefulset" resuming is not supported

檢查rollout的狀態

    $kubectl rollout status statefulset helloworld-statefulset
    partitioned roll out complete: 3 new pods have been updated...

## Rollback（倒回，降版）

將現有版本回朔到上一個版本。~~再度使用倒回，返回瑪莎多拉~~

    $kubectl rollout undo statefulsets.apps helloworld-statefulset
    statefulset.apps/helloworld-statefulset rolled back

好了應該長這樣

    $kubectl get po
    NAME                       READY   STATUS    RESTARTS   AGE
    helloworld-statefulset-0   1/1     Running   0          5m15s
    helloworld-statefulset-1   1/1     Running   0          5m48s
    helloworld-statefulset-2   1/1     Running   0          6m22s

檢查rollout狀態

    $kubectl rollout status statefulset helloworld-statefulset
    partitioned roll out complete: 3 new pods have been updated...

檢查rolling記錄

    $kubectl rollout history statefulset helloworld-statefulset
    statefulset.apps/helloworld-statefulset
    REVISION
    0
    0
    2
    3

以上為StatefulSet相關操作的流程，可以比較下與Deployment有什麼差別喔！

測試完畢後，我們就可以：

    $kubectl delete -f pod-statefulSet.yaml
    statefulset.apps "helloworld-statefulset" deleted

OK，測試完成！

# DaemonSet

這個部分前面也有稍微提過，我們再複習下：

DaemonSet能確保部份或所有node運行Pod的副本持續運行。
刪除DaemonSet將清除它創建的Pod。

我們看個例子：

    $vim pod-daemonSet.yaml
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: helloworld-daemonset
      labels:
        app: helloworld
    spec:
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
            image: 105552010/k8s-demo:v1
            ports:
            - containerPort: 3000

接著新增一個DaemonSet

    $kubectl create -f pod-daemonSet.yaml
    daemonset.apps/helloworld-daemonset created

檢查Pod狀態

    $kubectl get po
    NAME                         READY   STATUS    RESTARTS   AGE
    helloworld-daemonset-2j8j9   1/1     Running   0          11m

既然它保證持續運行，那我們來把這個Pod砍掉試試，嘿嘿～

    $kubectl delete po helloworld-daemonset-2j8j9
    pod "helloworld-daemonset-2j8j9" deleted

再次檢查Pod狀態

    $kubectl get po
    NAME                         READY   STATUS    RESTARTS   AGE
    helloworld-daemonset-4bhxj   1/1     Running   0          22s

顯然藉由DaemonSet，Pod就獲得了無限再生的技能。~~Pod表示：你殺了一個我，還有千千萬萬個我~~

我們來測試幾個操作：

## Update（更新，升版）

更新pod所用的image版本（將v1更新至v2

    $kubectl set image daemonset/helloworld-daemonset k8s-demo=105552010/k8s-demo:v2
    daemonset.apps/helloworld-daemonset image updated

檢查Pod狀態

    $kubectl get po
    NAME                         READY   STATUS        RESTARTS   AGE
    helloworld-daemonset-4bhxj   1/1     Terminating   0          7m28s
    $kubectl get po
    NAME                         READY   STATUS              RESTARTS   AGE
    helloworld-daemonset-54dvd   1/1     ContainerCreating   0          11s
    $kubectl get po
    NAME                         READY   STATUS    RESTARTS   AGE
    helloworld-daemonset-54dvd   1/1     Running   0          46s

與StatefulSet相同，不支援Pause和Resume

    $kubectl rollout pause daemonsets.apps helloworld-daemonset
    error: daemonsets.apps "helloworld-daemonset" pausing is not supported
    $kubectl rollout resume daemonsets.apps helloworld-daemonset
    error: daemonsets.apps "helloworld-daemonset" resuming is not supported

檢查rollout狀態

    $kubectl rollout status daemonset helloworld-daemonset
    daemon set "helloworld-daemonset" successfully rolled out

檢查rollout記錄

    $kubectl rollout history daemonset helloworld-daemonset
    daemonset.apps/helloworld-daemonset
    REVISION  CHANGE-CAUSE
    1         <none>
    2         <none>

## Rollback（倒回，降版）

將現有版本回朔到上一個版本。~~第三度使用倒回，返回瑪莎多拉，你是多想回家XD？~~

    $kubectl rollout undo daemonset helloworld-daemonset
    daemonset.apps/helloworld-daemonset rolled back

檢查Pod狀態

    $kubectl get po
    NAME                         READY   STATUS        RESTARTS   AGE
    helloworld-daemonset-54dvd   1/1     Terminating   0          8m39s
    $kubectl get po
    NAME                         READY   STATUS    RESTARTS   AGE
    helloworld-daemonset-dq2qq   1/1     Running   0          3s

檢查rolling狀態

    $kubectl rollout status daemonset helloworld-daemonset
    daemon set "helloworld-daemonset" successfully rolled out

檢查rolling記錄

    $kubectl rollout history daemonset helloworld-daemonset
    daemonset.apps/helloworld-daemonset
    REVISION  CHANGE-CAUSE
    2         <none>
    3         <none>

以上為DaemonSet相關操作的流程，也可以比較下與Deployment有什麼差別喔！

測試完畢後，我們也可以：

    $kubectl delete -f pod-daemonSet.yaml
    daemonset.apps "helloworld-daemonset" deleted

好的！測試結束了！！

# 小結

本日我們看了StatefulSet，它會給予Pod狀態，並提供標籤做唯一性的保證，在更新或其他操作的時候，是會按照順序的。再來我們看到了DaemonSet，它保證Pod是持續運行的狀態。當然，我們已經發現，在StatefulSet和DaemonSet的操作上，沒有什麼顯著的差異，但是在操作上，他們都跟Deployment有一些明顯的不同。隨著時間的推移，進階篇已經接近尾聲，明日是進階篇的最後一篇，後天就是要介紹管理篇了，敬請期待！我們明天見！

# 參考資料

- StatefulSet簡介
- DaemonSet簡介
