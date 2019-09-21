# 前言

昨天我們介紹了節點的架構，以及一些相關的簡易操作，我們來統整下昨天的觀念，也方便和今天的介紹作些比較，透過不同的面相來了解k8s，相信會比較輕鬆些。

# Node是基於clustering底下的單位，可分為Master和Slave

## Master

### kube-control-manager：

> k8s中負責管理的服務，它是基於kubernetes核心在運作的，也就是只要kubernetes有在運作，它會一直運行，能透過API Server來監控cluster

### kube-apiserver：

> k8s中提供監控的服務，自帶接口可與元件連接，像是Pod、Service、Controllers等，以REST作為operation定義了監控的執行方式，並提供前端分享cluster的狀態的方法來達成與元件互動的目的

### kube-scheduler：

> k8s中的排程器，因能夠獲取軟硬體及協定的資訊，所以會影響到整個master的效能，API Server能要求它給予額外的工作負載權限

## Slave

### kubelet：

> kubelet相當於 node agent，用於管理該 Node 上的所有 pods以及與 master node 即時溝通。

### kube-proxy：

> kube-proxy則是會將目前該 Node 上所有 Pods 的資訊傳給 iptables，讓 iptables 即時獲得在該 Node 上所有 Pod 的最新狀態。

# Kubernetes Objects

## Basic Object

### Pod：

> Pod是k8s中可以被創造或是發布的最小的單位，一個 Pod 在k8s世界中就相當於一個application。Pod有以下特點：
1. 每個 Pod 都有屬於自己的 yaml 檔
2. 一個 Pod 裡面可以包含一個或多個 Docker Container
3. 在同一個 Pod 裡面的 containers，可以用 local port 來互相溝通

### Service：

> k8s服務是一個abstraction，它定義了一組邏輯Pod和一個訪問它們的策略 - 有時稱為micro-service。負責不同Pod之間的溝通，舉例來說，像是前後端的服務需要互相溝通的時候，可以做同步異步的處理。Service就k8s而言是很重要的概念，有些細部的想法及設計請參考連結。

### Volume：

> k8s服務中的Volume與Docker的Volume是類似的概念，他們都是為了在虛擬化系統中，因該設計的pod或是container生命週期是短暫的，為了要保存資料的永久性和完整性所做的方法，也就是保存資料用的，也可以用作共享資料，無論是在不同的pod或是container之間，但是k8s的Volume不同於Docker的是，它會有明確的生命週期，且能在產生pod時加入屬於Volume不同的參數，像是要掛載的檔案系統類型等等（如：NFS），簡言之它具有更大的彈性，另外它可在container切換時保持存在，以達到交換資料的目的。

### Namespace：

> 所謂的Namespace，是劃分不同使用者能使用多少資源的方法（resource quota），舉例來說：CPU或是memory分配的多寡，在少量的使用者場景中（幾個或幾十個），這樣的功能是不需要的，此外，若是分配不同資源，並不需要使用不同的Namespace，只要在同個Namespace下定義不同的label即可。在管理k8s系統中的情況，最常使用到的是kube-system這個Namespace。

## High-level abstraction

### ReplicaSet：

> 複製集，是基於Replication Controller的改良版，本質差異是在selector的支援多寡，在現在的使用情境下，我們不會直接操作到ReplicaSet，每個不同的Deployment會有他們自己的ReplicaSet，並會自動調配，除非我們有對於複製集有特殊設定需求，不然一般直接套用Deployment的預設即可，ReplicaSet顧名思義是讓同個Pod有多個分身在同時運行，舉例來說：這樣的好處是在更新的時候不會讓使用者斷線，推薦閱讀：下方連結的Alternatives to ReplicaSet部分，該部分針對使用情境下有很詳細的解說。

### Deployment：

> 部署，Deployment Controller提供Pods和ReplicaSets可定義的更新，k8s下的Deployment有許多feature可以使用，如：create、update、roll back、pause、resume等，針對Deployment的替代方案可以使用kubectl rolling update這個指令。

### StatefulSet：

> StatefulSet管理一組Pod的部署（Deployment）和擴展（scaling），並提供有關這些Pod的排序和唯一性的保證。
與部署（Deployment）類似的是，StatefulSet管理基於相同容器規範的Pod。與部署（Deployment）不同的是，StatefulSet為其Pod保持標籤（label）。
這些Pod是根據相同的規範創建的，但不可互換：每個Pod都有一個永久性的標籤（label），它可以在任何重新安排時保留。

### DaemonSet：

> DaemonSet能確保部份或所有node運行Pod的副本持續運行。
刪除DaemonSet將清除它創建的Pod。

### Job：

> Job創建一個或多個pod，並確保指定數量的Pod工作完成後終止。
刪除Job將清除它創建的Pod。
Job還可以用於並行運行多個Pod。

# 小結

這邊我們除了複習了Node相關的內容外，也介紹了k8s的物件總共有哪些種類，以及大致上的使用情境為何，這篇主要篇整體概念為主，只要對一些基本名詞有些認知即可，細節的部分後面會陸續補上，主要是想先介紹整體的架構，細節的部分再慢慢談，先有個架構再去談細部結構我覺得會比較好，簡言之，我們可以看到：

[](https://www.notion.so/18f983ed5f4a42c4ade8cc7db2fc1f00#41724610b98348289e06476e2a69bc21)

以Node面向看k8s，Node有兩種，Master和Slave。~~跟斯斯有兩種一樣，一種治咳嗽，一種治鼻塞~~

[](https://www.notion.so/18f983ed5f4a42c4ade8cc7db2fc1f00#c9aab3c27ca04801a6c839d164b59435)

以Object面向看k8s，Object有兩種，Basic和Abstraction。

在後續理解了架構上的所有細節之後，可以再回到這裡對照一次，相信會有其他不同的體會。

先別太著重於細節，大致理解架構就好，加油！我們明天見！！

# 參考資料

- [kube controller manager介紹](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)
- [kube api server介紹](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
- [kube scheduler介紹](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)
- [kubelet介紹](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
- [kube proxy介紹](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)
- [Pod概觀](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)
- [Service概觀](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Volume概觀](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Namespace概觀](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [ReplicaSet概述](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Deployment概述](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [StatefulSet概述](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [DaemonSet概述](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [Job概述](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/)

本文同步刊載於[https://github.com/x1y2z3456/ironman](https://github.com/x1y2z3456/ironman)

感謝您撥冗閱讀此文章，不喜勿噴，有任何問題建議歡迎下方留言：）

說個笑話，希望我能寫滿30天啊（笑