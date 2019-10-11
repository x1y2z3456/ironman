# [Day26] k8s管理篇延伸與EKS（二）：Resource Quota、Namespace、RBAC

Author: Nick Zhuang
Type: kubernetes

# 前言

昨天我們看完了管理篇的上卷，今天我們要接續昨天的進度，繼續朝下卷邁進。

管理篇（上）：服務、監控（昨天）

管理篇（下）：資源控制、虛擬集群、使用者管理、RBAC（今天）

複習傳送門：

- [[Day14] k8s管理篇（一）：Monitoring、Job、CronJob](https://github.com/x1y2z3456/ironman/blob/master/day14/README.md)
- [[Day15] k8s管理篇（二）：Resource Quota、Namespaces](https://github.com/x1y2z3456/ironman/blob/master/day15/README.md)
- [[Day17] k8s管理篇（三）：User Management、RBAC、Node Ｍaintenance](https://github.com/x1y2z3456/ironman/blob/master/day17/README.md)

# 管理篇（下）

我們要先探討關於Resource Quota和NameSpace的整合範例

## 示例一：Resource Quota 和 Namespace

值得一提的是，EKS預設Resource Quota是開啟的，不用另外啟動，與minikube不同

我們先新增一個YAML

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

接著apply它

    $kubectl apply -f resourcequota.yaml
    namespace/myspace created
    resourcequota/compute-quota created
    resourcequota/object-quota created

### 沒有Resource Quota的Deployment

新增一個沒有設置Resource Quota的Deployment

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

接著apply它

    $kubectl apply -f deploy-no-quota.yaml -n myspace
    deployment.apps/helloworld-deployment created

檢查Pod狀態

    $kubectl get po -n myspace
    No resources found.

檢查Deployment狀態

    $kubectl describe deployment -n myspace
    Name:                   helloworld-deployment
    Namespace:              myspace
    CreationTimestamp:      Thu, 10 Oct 2019 17:46:02 +0800
    Labels:                 <none>
    Annotations:            deployment.kubernetes.io/revision: 1
                            kubectl.kubernetes.io/last-applied-configuration:
                              {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"helloworld-deployment","namespace":"myspace"},"spec":{"re...
    Selector:               app=helloworld
    Replicas:               3 desired | 0 updated | 0 total | 0 available | 3 unavailable
    StrategyType:           RollingUpdate
    MinReadySeconds:        0
    RollingUpdateStrategy:  25% max unavailable, 25% max surge
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
    Conditions:
      Type             Status  Reason
      ----             ------  ------
      Progressing      True    NewReplicaSetCreated
      Available        False   MinimumReplicasUnavailable
      ReplicaFailure   True    FailedCreate
    OldReplicaSets:    <none>
    NewReplicaSet:     helloworld-deployment-7865b8c744 (0/3 replicas created)
    Events:
      Type    Reason             Age   From                   Message
      ----    ------             ----  ----                   -------
      Normal  ScalingReplicaSet  90s   deployment-controller  Scaled up replica set helloworld-deployment-7865b8c744 to 3

這是因為沒有設置，所以不能調度

先把Deployment關掉

    $kubectl delete -f deploy-no-quota.yaml -n myspace
    deployment.apps "helloworld-deployment" deleted

### 有Resource Quota的Deployment

我們修改下Deployment的內容

    $vim deploy-with-quota.yaml
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

接著我們apply它

    $kubectl apply -f deploy-with-quota.yaml -n myspace
    deployment.apps/helloworld-deployment created

檢查Pod狀態

    $kubectl get po
    NAME                                     READY   STATUS    RESTARTS   AGE
    helloworld-deployment-5b5457694d-86pgn   1/1     Running   0          3h14m
    helloworld-deployment-5b5457694d-h5fbm   1/1     Running   0          3h14m

有了，要設置Resource Quota的Deployment才能用！

如果Namespace沒有要用的話，可以

    $kubectl delete namespaces myspace
    namespace "myspace" deleted

這樣就會把myspace底下所有相關的資源一併刪除。

OK，測試成功！

## 示例二：User management、RBAC

我們前面有講過EKS預設就已經將RBAC啟動

新增兩個檔案

- Private Key
```
    $vim ~/.aws/nick.key
    -----BEGIN RSA PRIVATE KEY-----
    MIIEpQIBAAKCAQEAo4/nVKTIeMt+byJq2RlRPwaQrpKaz3vv29leJbe72hh42sKK
    o+QTTWMVAVZ53Uh+slrAk/oIo7Fd4iAugHekMEUpPrbRo19rQ35HR7JS6GotyVcR
    4T0VuBPMb2ut18KZgA1rIjX67Jl2RQXolo8mEve09xH/uVVEr6QmcUBy3ms3lfgB
    Pv424JObt8SfRm0Km3ZSNcJTH4/wIl2FIDccibROvll6ws9sS/pdxOXcmD/N0kV9
    /qHn9l/6+oOqwvTh/3zHAfFzNlIgWDKo1cYe4A0Gl6oRKyObjR1ruB7oA3x9hOKy
    FJY9akO9IXiiOv/1FELhit5MD75BdzZm5nMs6wIDAQABAoIBAQCYTSpKLfqiSFJd
    5714lFOMDW/xrn/LDgvmOnypQHICfmEiyp3QSFMU71si2MQ3IgcfytOhtuQOkNzr
    7619YGqZq/zg8dk5eGNoAJEdGNaMpjomThZPFtM/iegGJE1HKGRI0bXdsEgLwkZX
    tU3DzF3WsaNnoPHvQI/pwT8um6WaptxwrFQP0gixaRAnylzRLrFNFvEDUPo8cEQq
    WgqAqctjufm+2+sm+0gO1+wdsPNrdHAoWxLs5MN8oqiZmtuFDoNA5g6usOv/aQNO
    wxpvCToIPUWg7fbCklSNftl4BFDJ9U/hFsXks0zbEvtKQZ3UrN+eqoE0I1Jo+y10
    8u0TCbnBAoGBANIAJppynsihrwLyMm953B/eazkNszBaMPlK1KkjnjjYPFTPWU1K
    a3KKCFmm4yISX5uIfeHmD/cugBeF0IZDEURc4Ia4purjP9o+t5QeWYUtLA7lniwh
    xu4rTAzhlSfV7EeC9M1/P3RG+w3cfOaFi5KThU2I9ynLLv3dwXZtv5dhAoGBAMdj
    s4WMDOeHmx+FISMi44GRBAq2RiAzr1ARKRV2syzSkNEvHUQF5P82+1iPXzYOQG5w
    8o6Xp593pQCUpaFEhogUXWuN+kIdlLOqYO2vhJ/347UbfVMbYzd9nc9Rtm/JuvEi
    8gVXE3MKz5tiWDQLoBqV8QnuMUpyPrOPL5IYwAPLAoGBAIDsCscC2yw85p6eZgw9
    +b+u4pCyMnHazPoe0JPOBBLN3awLZ72llHVK/HldlU+TjBKGJxIKFX8gkw7d3fiv
    L+iSRF0w+3h0bvzjR/ys7TRvWP8ERKi/S8tn1VaLHvDHyjjU0sld92zBLtuBo0Q6
    dEdWPZ4uGd8UmBLOkzjLg7XBAoGBAKqgbFklX0mm5x2THKdnzM7s3UuZbetSr3zS
    IplGidAapWkNa3rxnGS2lWLU1kJ48bRRHZDewMgbZ+1WR2L5NDMxUjyfNADuNXmG
    nQnpwJHwXUF3s8ix0DcFXU2z/G4vcLW4FOpy+KbjIoQzJY3sQOdfVvULi8zMdVHN
    f4UDfxX/AoGABniIKUWGs9xc+gPGPiNXXm9DZjU2XpzaYbXVPqcu2i3HSXUoxGbX
    /N8cIfqiVGVWIx0Kp69KNxZUMZYH0TeN02G5mCQ7Wasgf198FlCL+kVUR03ankEY
    cpdfqvvkZLaCMFOnJ88jj/vuzLo6zpoO8WDm1mN8ptzTcjQrNpUgEyo=
    -----END RSA PRIVATE KEY-----
```
- Certificate
```
    $vim ~/.aws/nick.crt
    -----BEGIN CERTIFICATE-----
    MIICsTCCAZkCCQC2dRsoRLLTozANBgkqhkiG9w0BAQUFADAVMRMwEQYDVQQDEwpt
    aW5pa3ViZUNBMB4XDTE5MDkzMDE3MzgwM1oXDTIyMDYyNjE3MzgwM1owIDENMAsG
    A1UEAwwEbmljazEPMA0GA1UECgwGbXl0ZWFtMIIBIjANBgkqhkiG9w0BAQEFAAOC
    AQ8AMIIBCgKCAQEAo4/nVKTIeMt+byJq2RlRPwaQrpKaz3vv29leJbe72hh42sKK
    o+QTTWMVAVZ53Uh+slrAk/oIo7Fd4iAugHekMEUpPrbRo19rQ35HR7JS6GotyVcR
    4T0VuBPMb2ut18KZgA1rIjX67Jl2RQXolo8mEve09xH/uVVEr6QmcUBy3ms3lfgB
    Pv424JObt8SfRm0Km3ZSNcJTH4/wIl2FIDccibROvll6ws9sS/pdxOXcmD/N0kV9
    /qHn9l/6+oOqwvTh/3zHAfFzNlIgWDKo1cYe4A0Gl6oRKyObjR1ruB7oA3x9hOKy
    FJY9akO9IXiiOv/1FELhit5MD75BdzZm5nMs6wIDAQABMA0GCSqGSIb3DQEBBQUA
    A4IBAQDD3yILyfHErcwhCvgiD4ODqqw5dmC7SH4m7YYXeRMIg2XW3YjRsP5Dww/M
    6rQtGkoSuiyJJs+lK0IZaPTSidHqe9Dfad5jNNUuMDkgClGlVmlT3EK7J7YvSbh0
    bIWzOfDBc+8OE0RRrR94jtYdpMxTf3p5M0iOJR8OS7822sv2aHylOEMPLt2fT/g7
    qgB2/nW3RzkTPDKbYm7IEqhL+eltIK/p+sNztDvbrxbKxXjauaHvVtMO23UP4dJD
    AZ0gZOm4zTF6mQykrcDm6m1zKWiaD+I0lhtuJ4+ZPZsZBI8w3Dmm+7ovoiT6hpbS
    oriyFXb/doOIIn0mTnNU/nyMYswW
    -----END CERTIFICATE-----
```
檢查既有的設定
```
    $kubectl config view
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority-data: DATA+OMITTED
        server: https://0E6EE730786C40D5356928C6B706CE71.yl4.ap-southeast-1.eks.amazonaws.com
      name: eks-cluster.ap-southeast-1.eksctl.io
    contexts:
    - context:
        cluster: eks-cluster.ap-southeast-1.eksctl.io
        user: eks@eks-cluster.ap-southeast-1.eksctl.io
      name: eks@eks-cluster.ap-southeast-1.eksctl.io
    current-context: eks@eks-cluster.ap-southeast-1.eksctl.io
    kind: Config
    preferences: {}
    users:
    - name: eks@eks-cluster.ap-southeast-1.eksctl.io
      user:
        exec:
          apiVersion: client.authentication.k8s.io/v1alpha1
          args:
          - token
          - -i
          - eks-cluster
          command: aws-iam-authenticator
          env: null
```
接著新增nick這個使用者
```
    $kubectl config set-credentials nick --client-certificate ~/.aws/nick.crt --client-key ~/.aws/nick.key
    User "nick" set.
    $kubectl config set-context nick --cluster=eks-cluster.ap-southeast-1.eksctl.io --user=nick
    Context "nick" created.
```
檢查更新的設定
```
    $kubectl config view
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority-data: DATA+OMITTED
        server: https://0E6EE730786C40D5356928C6B706CE71.yl4.ap-southeast-1.eks.amazonaws.com
      name: eks-cluster.ap-southeast-1.eksctl.io
    contexts:
    - context:
        cluster: eks-cluster.ap-southeast-1.eksctl.io
        user: eks@eks-cluster.ap-southeast-1.eksctl.io
      name: eks@eks-cluster.ap-southeast-1.eksctl.io
    **- context:
        cluster: eks-cluster.ap-southeast-1.eksctl.io
        user: nick
      name: nick**
    current-context: eks@eks-cluster.ap-southeast-1.eksctl.io
    kind: Config
    preferences: {}
    users:
    - name: eks@eks-cluster.ap-southeast-1.eksctl.io
      user:
        exec:
          apiVersion: client.authentication.k8s.io/v1alpha1
          args:
          - token
          - -i
          - eks-cluster
          command: aws-iam-authenticator
          env: null
    **- name: nick
      user:
        client-certificate: /home/nick/.aws/nick.crt
        client-key: /home/nick/.aws/nick.key**
```
顯示當前使用者
```
    $kubectl config current-context
    eks@eks-cluster.ap-southeast-1.eksctl.io
```
切換到nick
```
    $kubectl config use-context nick
    Switched to context "nick".
```
顯示當前使用者
```
    $kubectl config current-context
    nick
```
使用者nick操作集群
```
    $kubectl get po
    error: the server doesn't have a resource type "po"
```
這是還沒有設定權限造成的

接著我們幫nick新增權限

新增一個Role

    $vim role.yaml
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      namespace: default
      name: pod-reader
    rules:
    - apiGroups: [""] # "" indicates the core API group
      resources: ["pods"]
      verbs: ["get", "watch", "list"]

新增一個RoleBinding

    $vim role-binding.yaml
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: read-pods
      namespace: default
    subjects:
    - kind: User
      name: nick # Name is case sensitive
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: Role #this must be Role or ClusterRole
      name: pod-reader # must match the name of the Role
      apiGroup: rbac.authorization.k8s.io

先切回AWS的EKS管理員

    $kubectl config use-context eks@eks-cluster.ap-southeast-1.eksctl.io
    Switched to context "eks@eks-cluster.ap-southeast-1.eksctl.io".

接著啟動Role和RoleBinding

    $kubectl apply -f role.yaml
    role.rbac.authorization.k8s.io/pod-reader created
    $kubectl apply -f role-binding.yaml
    rolebinding.rbac.authorization.k8s.io/read-pods created

檢查Role和RoleBinding的狀態

    $kubectl get role
    NAME         AGE
    pod-reader   87s
    $kubectl get rolebinding
    NAME        AGE
    read-pods   56s

詳細看Role和RoleBinding的狀態

    $kubectl describe role
    Name:         pod-reader
    Labels:       <none>
    Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                    {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"Role","metadata":{"annotations":{},"name":"pod-reader","namespace":"default"},"rules"...
    PolicyRule:
      Resources  Non-Resource URLs  Resource Names  Verbs
      ---------  -----------------  --------------  -----
      pods       []                 []              [get watch list]
    $kubectl describe rolebinding
    Name:         read-pods
    Labels:       <none>
    Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                    {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"RoleBinding","metadata":{"annotations":{},"name":"read-pods","namespace":"default"},"...
    Role:
      Kind:  Role
      Name:  pod-reader
    Subjects:
      Kind  Name  Namespace
      ----  ----  ---------
      User  nick

有看到nick這個使用者註冊上去了！

接著我們切換使用者，換成nick試試

    $kubectl config current-context
    eks@eks-cluster.ap-southeast-1.eksctl.io
    $kubectl config use-context nick
    Switched to context "nick".
    $kubectl config current-context
    nick
    $kubectl get po
    error: You must be logged in to the server (Unauthorized)

唷！沒有授權，這應該是AWS的關係

我們修改下~/.kube/config，增加exec的Block內容

    $vim ~/.kube/config
    - name: nick
      user:
        **exec:
          apiVersion: client.authentication.k8s.io/v1alpha1
          args:
          - token
          - -i
          - eks-cluster
          command: aws-iam-authenticator
          env: null**
        client-certificate: /home/nick/.aws/nick.crt
        client-key: /home/nick/.aws/nick.key

這個是讓能使用aws-iam-authenticator去認證

接著我們在操作一次

    $kubectl get po
    No resources found.

可以用了，我們把它恢復原狀

    $kubectl delete -f role.yaml
    role.rbac.authorization.k8s.io "pod-reader" deleted
    $kubectl delete -f role-binding.yaml
    rolebinding.rbac.authorization.k8s.io "read-pods" deleted
    $kubectl get role
    No resources found.
    $kubectl get rolebindings.rbac.authorization.k8s.io
    No resources found.

測試結束。

# 小結

今天我們已經將AWS上的管理篇內容做個結束，相信讀者有不少體會，隨著30天邁入尾聲，我們會開始介紹AWS上的k8s應用篇，以及一些相關群集整體的內容，有些整合性的內容會開始放上來，敬請期待，我們明天見囉～

# 參考資料

- [Resource Quota簡介](https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/)
- [Namespace簡介](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [k8s基於RBAC的用戶管理](https://medium.com/@HoussemDellai/rbac-with-kubernetes-in-minikube-4deed658ea7b)
- [k8s的RBAC授權](https://nicksors.cc/2019/07/09/kubernetes%E7%B3%BB%E5%88%97%E4%B9%8B%E3%80%8Ak8s%E5%9F%BA%E4%BA%8ERBAC%E7%9A%84%E6%8E%88%E6%9D%83%E3%80%8B.html)
- [k8s的授權](https://kubernetes.io/zh/docs/reference/access-authn-authz/authorization/)
- [Unauthorized排錯](https://github.com/kubernetes-sigs/aws-iam-authenticator/issues/105)
- [AWS新增RBAC的User詳細步驟](https://eksworkshop.com/intro_to_rbac/)
