# 第十九天：k8s之十二：上雲端之路：AWS之蓄勢待發：EOO

Author: Nick Zhuang
Type: kubernetes

# 前言

今天我們要延續之前的內容，在上雲端之前，先介紹一下CCM，所謂的CCM是Cloud Controller Manager，在雲端服務上它是基於k8s集群上的一個管理器，~~QOO有種果汁真好喝~~EOO元件介紹AWS，是一些在AWS的基本元件，如：EC2、EBS、ELB、EFS、EKS、ENI、ECR，接著是EKS和KOPS的比較，說明為何要用EKS。最後是一些使用AWS的前置準備，要先設置下。

# CCM

雲控制器管理器（cloud controller manager，CCM）這個概念創建的初衷是為了讓特定的雲服務供應商代碼和k8s核心相互獨立演化。雲控制器管理器與其他主要組件（如k8s控制器管理器，API服務器和調度程序）一起運行。它也可以作為k8s的插件啟動，在這種情況下，它會運行在k8s之上。

在沒有這樣的設置下系統大致長這樣：

## 無CCM的架構

![](https://d33wubrfki0l68.cloudfront.net/e298a92e2454520dddefc3b4df28ad68f9b91c6f/70d52/images/docs/pre-ccm-arch.png)

這張圖我們之前有看過，就是這篇：[Master與Slave的不同](https://www.notion.so/k8s-Node-Master-Slave-1d2f16636b924a8fa2f7959f171e129a)，有提過相同的圖片，這是沒有上雲的情況

接著我們再看一張

## 有CCM的架構

![](https://d33wubrfki0l68.cloudfront.net/518e18713c865fe67a5f23fc64260806d72b38f5/61d75/images/docs/post-ccm-arch.png)

這個是有上雲的情況，可以發現到：

- 對於單節點那些部分的連接全部轉換到kube-apiserver上
- 增加了cloud-controller manager，並協調kube-controller manager及連接kube-apiserver

CCM 打破了k8s控制器管理器（KCM）的一些功能，並將其作為一個單獨的進程運行。具體來說，它打破了KCM中依賴於雲的控制器。

注意卷控制器不屬於CCM，由於其中涉及到的複雜性和對現有供應商特定卷的邏輯抽象，因此決定了卷控制器不會被移動到CCM 之中。

CCM可以設置ClusterRole以達成授權，這部分與介紹RBAC時類似

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: cloud-controller-manager
    rules:
    - apiGroups:
      - ""
      resources:
      - nodes
      verbs:
      - '*'
    - apiGroups:
      - ""
      resources:
      - nodes/status
      verbs:
      - patch
    - apiGroups:
      - ""
      resources:
      - services
      verbs:
      - list
      - patch
      - update
      - watch
    - apiGroups:
      - ""
      resources:
      - serviceaccounts
      verbs:
      - create
    - apiGroups:
      - ""
      resources:
      - persistentvolumes
      verbs:
      - get
      - list
      - update
      - watch

這部分等到上雲的時候，我們會再測試。

# AWS的元件

筆者後續的介紹都以AWS為主，如果要找其他的，請參考：[雲端服務商](https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/)

## EC2、EBS、ELB、EFS、EKS、ENI、ECR

這些元件是我們在用EKS架設k8s群集的時候會用到了，大致介紹，設置的時候會再提。

## EC2

Elastic Compute Cloud（EC2） ，是由[亞馬遜公司](https://zh.wikipedia.org/wiki/%E4%BA%9E%E9%A6%AC%E9%81%9C%E5%85%AC%E5%8F%B8)提供的[Web服務](https://zh.wikipedia.org/wiki/Web%E6%9C%8D%E5%8B%99)，是一個讓使用者可以租用[雲端](https://zh.wikipedia.org/wiki/%E9%9B%B2%E7%AB%AF%E9%81%8B%E7%AE%97)電腦運行所需應用的系統。EC2藉由提供Web服務的方式讓使用者可以彈性地運行自己的機器映像檔，使用者將可以在這個[虛擬機器](https://zh.wikipedia.org/wiki/%E8%99%9B%E6%93%AC%E6%A9%9F%E5%99%A8)上運行任何自己想要的軟體或應用程式，也可以隨時建立、執行、終止自己的虛擬伺服器，使用多少時間算多少錢，也因此這個系統是「彈性」使用的。EC2讓使用者可以控制執行虛擬伺服器的主機地理位置，這可以改善讓延遲還有備援性例如，為了讓系統維護時間最短，用戶可以在每個時區都運行自己的虛擬伺服器。

## EBS

Elastic Block Store (EBS) 是易於使用的高效能區塊儲存服務，專為與Elastic Compute Cloud (EC2) 搭配使用而設計，能以任何規模同時用於輸送量和交易密集型工作負載。各種工作負載 (如關聯式和非關聯式資料庫、企業應用程式、容器化的應用程式、大數據分析引擎、檔案系統和媒體工作流程) 廣泛部署在 EBS 上。

## ELB

Elastic Load Balancing 可在多個目標 (例如 EC2 執行個體、容器、IP 地址和 Lambda 函數) 之間自動分配傳入的應用程式流量。它可以在單一可用區域或跨多個可用區域處理應用程式流量的各種負載。Elastic Load Balancing 提供三種負載平衡器，它們都具有下列特性：高可用性、自動擴展，以及讓應用程式具備容錯功能的強大安全防護。

## EFS

Elastic File System (EFS) 提供簡單、可擴展、全受管的彈性 NFS 檔案儲存，可與 AWS 雲端服務和內部部署資源搭配使用。其建置是為了要隨需擴展至數 PB，且不會中斷應用程式，可隨著新增和移除檔案自動擴展和縮減，無須佈建和管理容量來適應增長。

EFS 提供兩種儲存類別：標準儲存類別和[不常存取儲存類別](https://aws.amazon.com/tw/efs/features/infrequent-access/) (EFS IA)。EFS IA 針對每天未存取的檔案提供成本優化的性價比。

## EKS

Elastic Kubernetes Service (EKS) 可使用 [AWS 上的 Kubernetes](https://aws.amazon.com/tw/kubernetes/) 輕鬆部署、管理和擴展容器化應用程式。

EKS可在多個 AWS 可用區域為執行 Kubernetes 管理基礎設施，以避免發生單一故障點問題。 EKS 已取得 Kubernetes 一致性授權，可讓使用合作夥伴和 Kubernetes 社群的現有工具和外掛程式。在任何標準 Kubernetes 環境執行的應用程式都能完全相容，而且可以輕鬆遷移到 EKS。

## ENI

Elastic Network Interface（ENI），彈性網路界面是代表虛擬網路卡之 VPC 中的邏輯聯網元件。

可在帳戶中建立並設定網路界面，然後將它們連接到VPC 中的執行個體。

## ECR

Elastic Container Registry (ECR) 是一個全受管的 Docker 容器登錄檔，可讓開發人員輕鬆存放、管理以及部署 Docker 容器映像。ECR 已與 Elastic Container Service (ECS) 整合，以簡化從開發到生產的工作流程。

# EKS與Kops的比較

看完了上面的介紹，我們大致可以知道，EKS是在AWS上部署k8s可以用的工具，不過他只能用在AWS上，Kops同樣也是部署k8s的工具，不過它是開源的，可以適用於不同的雲環境，雖然它的泛用性比較高，但是相較於EKS有些不足的地方，Kops出來很久了，發展算是很完整，有相當程度的穩定性，也因此可以依賴。EKS是比較新的東西，大概出來1年多左右而已，我們來看張比較表

|項目      |EKS   |Kops  |
|---------|------|------|
|集群設置   |較差  |較好   |
|集群管理   |較好  |較差   |
|安全性     |較好  |較差   |

所以來個結論，Kops在設置上有更多可以操作的空間，但是也相對複雜，整體來說，EKS可能比較適合軟件開發人員，而Kops比較適合真正的系統管理者。

# 前置準備

要使用AWS服務，必須先在他們網站登錄一個新帳號，會需要用到Email認證。

好了之後，先登入到AWS

![](_2019-10-03_12-ef96ddb3-a701-4684-84aa-38ccbb2c18c5.55.30.png)

## 新增AWS Account的User

進去之後，應該有如下類似畫面，選擇IAM的Service

![](_2019-10-03_1-392d98de-b934-4a8e-8bc3-239f2c051cf1.42.07.png)

選好後理應顯示如下

![](_2019-10-03_1-d1f16754-be8c-4b57-b18a-6b23c9ab14b1.13.01.png)

點選Users後，選擇Add user

![](_2019-10-03_1-0b8494b8-cf52-44fd-a14f-fc8a6797befd.49.35.png)

使用者名稱：eks，一些Access type勾一勾，接著下一步

![](_2019-10-03_1-21d87e46-29e4-46a5-bafb-8e4666864f36.47.05.png)

因為我們這邊還沒有Group，每個使用者都要綁定一個Group，這類似於Linux系統本身的設計

我們先Create group

![](_2019-10-03_1-b03d018f-f9eb-4d23-8f21-393b9e7d2b9f.52.17.png)

Group name隨意取，筆者是用eksGroup，但要注意的是，一定要有下面的AdministratorAccess，好了就按Create group

![](_2019-10-03_2-f7615f5c-692a-48c8-b0a2-fdc30f7b41ee.20.19.png)

這個時候就可以看到在這個頁面有新的Group，名為eksGroup，接著我們按"Next: Tags"進到下一步

![](_2019-10-03_2-d2fe60d7-6063-4eb5-9519-d77e8d899dea.20.39.png)

他這裡是問你說要不要上Tag，是否要把使用者加上標籤，可以不用，進到下一步"Next: Review"

![](_2019-10-03_1-21360055-61ae-457c-9664-0b5c7db0d677.53.56.png)

這裡是檢視整個使用者的相關訊息，確認後就會按照這個設定新增使用者

![](_2019-10-03_2-08228f39-5f31-427b-a22a-872034b337ac.21.34.png)

最後我們檢查下

![](_2019-10-03_2-890b4307-4508-4985-9a8f-87f67a0a2c15.33.33.png)

OK，使用者eks建立完成！

## 新增IAM Role

接著我們切換到Roles

![](_2019-10-03_2-19c1c8e9-1104-42e9-afe3-ed7d6a5e689a.57.11.png)

在這邊我們要選擇新增EKS的Role，然後"Next: Permissions"

![](_2019-10-03_3-d3caf29f-a08a-4e01-8ffa-fa111cd210ad.06.53.png)

接著Tag設定一下

![](_2019-10-03_3-d3ad31a1-bcdc-4836-9f1e-0fadb37964cc.13.50.png)

接著檢視要新增的Role資訊

![](_2019-10-03_3-9d7d9992-89a8-4734-908e-8773fb7c9919.15.55.png)

最後檢視一下新增Role的結果

![](_2019-10-03_3-4d5c0b24-2b0a-473b-af70-9976903f1aa3.16.45.png)

OK，IAM Role建立完成！

## 在EC2新增Key Value的Pair

回到Management Service的介面，選擇EC2

![](_2019-10-03_3-f629f2b2-6871-40af-8479-2c1af04fcb7b.23.21.png)

在EC2中新增Key Value的Pair

![](_2019-10-03_3-456c297d-ef49-4659-9c6d-b027cfd41147.27.33.png)

新增Key Value的Pair

![](_2019-10-03_3-aa5be2c9-3769-4642-9548-d20624035dbc.33.28.png)

檢視結果

![](_2019-10-03_3-5a6ffb5e-875a-4125-87ac-2018f18107a5.30.21.png)

接著回到IAM，產生新的Access Key

![](_2019-10-03_3-f69a2635-6d41-46e8-8ef1-21d37fb6a0d2.43.53.png)

產生key後可以下載對應的csv檔，它那個是Access Key，另外還有一個pem檔是SSH Key，兩個都是私鑰，請妥善保管

![](_2019-10-03_3-a95a0840-aa48-40cc-b3de-5f2a92a751e4.45.36.png)

# 小結

今天我們看過了CCM，這是Cloud Compute Manager，在雲端上對於集群是透過它去操作的。再來我們看到了一些要在AWS上架設k8s集群所需要用到的一些AWS元件，如：EC2、EBS、ELB、EFS、EKS、ENI、ECR等等，這些元件屆時上雲的時候會有詳細的設置解說，再來我們看到了在雲端上架設k8s集群的工具，像是EKS、Kops，我們做了一些比較，這是整理前人的經驗，後續我們在使用EKS的時候，相信會有更深的體會，最後我們針對AWS有做了一些前置準備，明天會開始使用EKS架設k8s群集，我們明天見囉～

# 參考資料

- [CCM](https://kubernetes.io/zh/docs/concepts/architecture/cloud-controller/)
- [AWS EC2](https://zh.wikipedia.org/zh-tw/Amazon_EC2)
- [AWS ELB](https://aws.amazon.com/tw/elasticloadbalancing/)
- [AWS EBS](https://aws.amazon.com/tw/ebs/)
- [AWS EFS](https://aws.amazon.com/tw/efs/?nc2=h_m1)
- [AWS EKS](https://aws.amazon.com/tw/eks/)
- [AWS ENI](https://docs.aws.amazon.com/zh_tw/AWSEC2/latest/UserGuide/using-eni.html)
- [AWS ECR](https://aws.amazon.com/tw/ecr/)
- [EKS與Kops的比較](https://www.bluematador.com/blog/kubernetes-on-aws-eks-vs-kops)
