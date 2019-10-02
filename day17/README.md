# 第十七天：k8s之十：管理篇之三，User Management、RBAC、Node Ｍaintenance

Author: Nick Zhuang
Type: kubernetes

# 前言

今天是管理篇的最後一篇，我們最後要介紹User Management和RBAC，就是使用者的權限操作，如：新增使用者、設置對應權限，以及Node Maintenance，也就是相對應的集群節點管理方式，底下詳細介紹。

# User Management與RBAC

這邊我們直接給範例，來快速測試一下，會比較好了解

## 新增一個New User

這邊的目標是，要在集群內新增nick這個使用者

我們先登入minikube的ssh

    $minikube ssh
                             _             _
                _         _ ( )           ( )
      ___ ___  (_)  ___  (_)| |/')  _   _ | |_      __
    /' _ ` _ `\| |/' _ `\| || , <  ( ) ( )| '_`\  /'__`\
    | ( ) ( ) || || ( ) || || |\`\ | (_) || |_) )(  ___/
    (_) (_) (_)(_)(_) (_)(_)(_) (_)`\___/'(_,__/'`\____)
    
    $

新增一個RSA的金鑰

    $ openssl genrsa -out nick.pem 2048
    Generating RSA private key, 2048 bit long modulus
    ...................................+++
    ..........+++
    e is 65537 (0x10001)
    $ ls
    nick.pem

產生該金鑰對應的Certificate

X.509是密碼學裡公鑰憑證的格式標準

    $ openssl req -new -key nick.pem -out nick-csr.pem -subj "/CN=nick/O=myteam"
    $ ls
    nick-csr.pem  nick.pem
    $ sudo openssl x509 -req -in nick-csr.pem -CA /var/lib/minikube/certs/ca.crt -CAkey /var/lib/minikube/certs/ca.key -CAcreateserial -out nick.crt -days 1000
    Signature ok
    subject=/CN=nick/O=myteam
    Getting CA Private Key
    $ ls
    nick-csr.pem  nick.crt  nick.pem

檢查Certificate

    $ cat nick.crt
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

檢查私鑰

    $ cat nick.pem
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

接著跳出ssh，回到一般命令列操作

新增~/.minikube/nick.crt（就是nick.crt的內容

    $vim ~/.minikube/nick.crt
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

新增~/.minikube/nick.key（就是nick.pem的內容

    $vim ~/.minikube/nick.key
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

新增好這兩個檔案後，我們要下些指令：

首先我們先檢查已知的k8s的config

    $kubectl config view
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority: /Users/nickzhuang/.minikube/ca.crt
        server: https://192.168.99.100:8443
      name: minikube
    contexts:
    - context:
        cluster: minikube
        user: minikube
      name: minikube
    current-context: minikube
    kind: Config
    preferences: {}
    users:
    - name: minikube
      user:
        client-certificate: /Users/nickzhuang/.minikube/apiserver.crt
        client-key: /Users/nickzhuang/.minikube/apiserver.key

這個內容跟~/.kube/config的內容一樣喔，可以自行檢查看看！

接著我們利用前面產生的Certificate和Key去新增使用者資訊

    $kubectl config set-credentials nick --client-certificate ~/.minikube/nick.crt --client-key ~/.minikube/nick.key
    User "nick" set.
    $kubectl config set-context nick-context --cluster=minikube --user=nick
    Context "nick-context" created.

再次檢查k8s的config，發現到Context和User區塊有增加內容

    $kubectl config view
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority: /Users/nickzhuang/.minikube/ca.crt
        server: https://192.168.99.100:8443
      name: minikube
    contexts:
    - context:
        cluster: minikube
        user: minikube
      name: minikube
    - context:
        cluster: minikube
        user: nick
      name: nick-context
    current-context: minikube
    kind: Config
    preferences: {}
    users:
    - name: minikube
      user:
        client-certificate: /Users/nickzhuang/.minikube/apiserver.crt
        client-key: /Users/nickzhuang/.minikube/apiserver.key
    - name: nick
      user:
        client-certificate: /Users/nickzhuang/.minikube/nick.crt
        client-key: /Users/nickzhuang/.minikube/nick.key

檢查當前的Context，Context對應到目前的使用者，應該是minikube

    $kubectl config current-context
    minikube

切換使用者到nick

    $kubectl config use-context nick-context
    Switched to context "nick-context".

檢視集群節點狀態

    $kubectl get po
    Error from server (Forbidden): pods is forbidden: User "nick" cannot list resource "pods" in API group "" in the namespace "default"

它這裡是說沒有權限，不用擔心，因為我們還沒有設置，請接續下個部分

## Role與RoleBinding

目前k8s支援四種模式，分別是：

- `Node`：一個專用授權程序，根據計劃運行的pod為kubelet授予權限。
- `ABAC`：基於屬性的訪問控制（Attribute-based access control），是一種通過將屬性組合在一起的策略，將訪問權限授予用戶的方法。
- `RBAC`：基於角色的訪問控制（Role-based access control），是一種基於用戶的角色，來管理對計算機或網絡資源的訪問的方法。
- `Webhook`：WebHook是一個HTTP回調，發生某些事情時會通過HTTP POST進行簡單的事件通知。

下面示範一種常用的模式RBAC，這種設定可以滿足讓不同使用者擁有不同權限的需求。

## 將New User綁定RBAC

延續上個部分，在這邊我們要先新增Role和RoleBinding

附帶一提，minikube要使用RBAC，必須要在集群啟動時啟用設定

    $minikube start —-extra-config=apiserver.Authorization.Mode=RBAC

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

因為在上個例子，我們已經切換到了nick的身份，但是因為他沒有權限，所以我們要先切換回minikube，也就是Administrator的身份

    $kubectl config use-context minikube
    Switched to context "minikube".

接著啟動Role和RoleBinding

    $kubectl apply -f role.yaml
    role.rbac.authorization.k8s.io/pod-reader created
    $kubectl apply -f role-binding.yaml
    rolebinding.rbac.authorization.k8s.io/read-pods created

檢查Role和RoleBinding的狀態

    $kubectl get role
    NAME         AGE
    pod-reader   30s
    $kubectl get rolebindings
    NAME        AGE
    read-pods   72s

詳細檢查Role和RoleBinding

    $kubectl describe role pod-reader
    Name:         pod-reader
    Labels:       <none>
    Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                    {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"Role","metadata":{"annotations":{},"name":"pod-reader","namespace":"default"},"rules"...
    PolicyRule:
      Resources  Non-Resource URLs  Resource Names  Verbs
      ---------  -----------------  --------------  -----
      pods       []                 []              [get watch list]
    $kubectl describe rolebindings.rbac.authorization.k8s.io read-pods
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

有的，可以看到New User：nick的狀態。

接著我們切換使用者到nick

    $kubectl config use-context nick-context
    Switched to context "nick-context".

然後我們嘗試獲取集群的Pod資訊

    $kubectl get po
    No resources found.

可以用了！現在使用者nick可以獲得集群的Pod資訊

測試完沒用到的可以刪掉

    $kubectl delete -f role.yaml
    role.rbac.authorization.k8s.io "pod-reader" deleted
    $kubectl delete -f role-binding.yaml
    rolebinding.rbac.authorization.k8s.io "read-pods" deleted
    $kubectl get role
    No resources found.
    $kubectl get rolebindings.rbac.authorization.k8s.io
    No resources found.

回復原狀囉～

# Node Ｍaintenance

這個部分前面有稍微提過，就是關於[k8s管理篇初體驗](https://ithelp.ithome.com.tw/articles/10218001)那邊，我們有稍稍操作到了一些管理Node的方法，這邊我們更進一步，來測試一下在Node管理階段的時候，在該Node上的元件會有哪些預期行為。

## 管理三劍客：Drain、Cordon、Uncordon

我們先編輯一個Deployment的YAML

    $vim pod-deploy.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: helloworld-deployment
      labels:
        app: helloworld
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
            image: 105552010/k8s-demo:v1
            ports:
            - containerPort: 3000

接著新增它到集群裡

    $kubectl create -f pod-deploy.yaml
    deployment.apps/helloworld-deployment created

檢查Pod狀態

    $kubectl get po
    NAME                                     READY   STATUS              RESTARTS   AGE
    helloworld-deployment-6777fcc7f4-5vtn6   0/1     ContainerCreating   0          4s
    helloworld-deployment-6777fcc7f4-99478   0/1     ContainerCreating   0          4s
    helloworld-deployment-6777fcc7f4-nx9q7   0/1     ContainerCreating   0          4s
    $kubectl get po
    NAME                                    READY   STATUS              RESTARTS   AGE
    helloworld-deployment-94d865798-d8mnx   1/1     Running             0          5s
    helloworld-deployment-94d865798-ffn5c   0/1     ContainerCreating   0          5s
    helloworld-deployment-94d865798-m6tp6   1/1     Running             0          5s
    $kubectl get po
    NAME                                    READY   STATUS    RESTARTS   AGE
    helloworld-deployment-94d865798-d8mnx   1/1     Running   0          10s
    helloworld-deployment-94d865798-ffn5c   1/1     Running   0          10s
    helloworld-deployment-94d865798-m6tp6   1/1     Running   0          10s

OK，啟動正常，我們接續下一步

接著我們將Node設為不可調度

    $kubectl drain minikube --ignore-daemonsets --delete-local-data
    node/minikube already cordoned
    WARNING: ignoring DaemonSet-managed Pods: kube-system/kube-proxy-pj4df
    evicting pod "dashboard-metrics-scraper-6c554969c6-g5ftb"
    evicting pod "helloworld-deployment-94d865798-m6tp6"
    evicting pod "dashboard-metrics-scraper-6c554969c6-4jqkw"
    evicting pod "dashboard-metrics-scraper-6c554969c6-fd7bg"
    evicting pod "kubernetes-dashboard-74f5d74fd7-vkx8q"
    evicting pod "kubernetes-dashboard-74f5d74fd7-fbgd2"
    evicting pod "kubernetes-dashboard-74f5d74fd7-mcblb"
    evicting pod "kubernetes-dashboard-74f5d74fd7-t9zv9"
    evicting pod "helloworld-deployment-94d865798-d8mnx"
    evicting pod "helloworld-deployment-94d865798-ffn5c"
    evicting pod "kubernetes-dashboard-74f5d74fd7-c9f9t"
    evicting pod "dashboard-metrics-scraper-6c554969c6-wkwvx"
    evicting pod "kubernetes-dashboard-74f5d74fd7-7hlkv"
    evicting pod "coredns-5644d7b6d9-smn5n"
    evicting pod "coredns-5644d7b6d9-z6zgs"
    evicting pod "storage-provisioner"
    pod/kubernetes-dashboard-74f5d74fd7-c9f9t evicted
    pod/dashboard-metrics-scraper-6c554969c6-g5ftb evicted
    pod/dashboard-metrics-scraper-6c554969c6-4jqkw evicted
    pod/kubernetes-dashboard-74f5d74fd7-7hlkv evicted
    pod/dashboard-metrics-scraper-6c554969c6-wkwvx evicted
    pod/kubernetes-dashboard-74f5d74fd7-fbgd2 evicted
    pod/kubernetes-dashboard-74f5d74fd7-mcblb evicted
    pod/kubernetes-dashboard-74f5d74fd7-t9zv9 evicted
    pod/storage-provisioner evicted
    pod/dashboard-metrics-scraper-6c554969c6-fd7bg evicted
    pod/kubernetes-dashboard-74f5d74fd7-vkx8q evicted
    pod/coredns-5644d7b6d9-smn5n evicted
    pod/coredns-5644d7b6d9-z6zgs evicted
    pod/helloworld-deployment-94d865798-ffn5c evicted
    pod/helloworld-deployment-94d865798-m6tp6 evicted
    pod/helloworld-deployment-94d865798-d8mnx evicted
    node/minikube evicted

注意到我們之前的做法是用cordon，drain與cordon不同的是，它會在集群內找尋其他可調用的Node，並將終止的那些Pod轉移過去

不過因為現在是單節點的集群，所以看不出轉移的部分，這個部分我們會在上雲的時候再看一次，就能看出差異。

我們來看下剛剛那些Pod

    $kubectl get po
    NAME                                    READY   STATUS    RESTARTS   AGE
    helloworld-deployment-94d865798-47gxp   0/1     Pending   0          3m7s
    helloworld-deployment-94d865798-rv729   0/1     Pending   0          3m7s
    helloworld-deployment-94d865798-vshxc   0/1     Pending   0          3m7s

發現到都是Pending的狀態

檢查集群Node

    $kubectl get nodes
    NAME       STATUS                     ROLES    AGE   VERSION
    minikube   Ready,SchedulingDisabled   master   16h   v1.16.0

這個應用情境是當機器需要定期保養、維護的時候，可以先停機，等到機器恢復了，就可以繼續用

    $kubectl uncordon minikube
    node/minikube uncordoned
    $kubectl get nodes
    NAME       STATUS   ROLES    AGE   VERSION
    minikube   Ready    master   16h   v1.16.0

檢查Pod狀態，顯然已經恢復正常

    $kubectl get po
    NAME                                    READY   STATUS    RESTARTS   AGE
    helloworld-deployment-94d865798-47gxp   1/1     Running   0          6m3s
    helloworld-deployment-94d865798-rv729   1/1     Running   0          6m3s
    helloworld-deployment-94d865798-vshxc   1/1     Running   0          6m3s

我們把它cordon試試

    $kubectl cordon minikube
    node/minikube cordoned

發現Pod狀態還是沒變（已經調度的情況不影響，但後續要調度會失敗

    $kubectl get po
    NAME                                    READY   STATUS    RESTARTS   AGE
    helloworld-deployment-94d865798-47gxp   1/1     Running   0          6m27s
    helloworld-deployment-94d865798-rv729   1/1     Running   0          6m27s
    helloworld-deployment-94d865798-vshxc   1/1     Running   0          6m27s

新增一個Pod

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
        - touch /tmp/healthy; npm start; sleep 60; rm -rf /tmp/healthy; sleep 600
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

接著調度它

    $kubectl create -f pod-demo.yaml
    pod/my-helloworld created

檢查集群的Pod狀態

    $kubectl get po
    NAME                                    READY   STATUS    RESTARTS   AGE
    helloworld-deployment-94d865798-47gxp   1/1     Running   0          8m54s
    helloworld-deployment-94d865798-rv729   1/1     Running   0          8m54s
    helloworld-deployment-94d865798-vshxc   1/1     Running   0          8m54s
    my-helloworld                           0/1     Pending   0          4s

它會一直維持Pending，因為Node已經不能調度了。

測試成功！~~小屁孩再度把玩耍的恢復原狀。~~

    $kubectl delete -f pod-demo.yaml
    pod "my-helloworld" deleted
    $kubectl delete -f pod-deploy.yaml
    deployment.apps "helloworld-deployment" deleted
    $kubectl uncordon minikube
    node/minikube uncordoned

檢查集群狀態

    $kubectl get all
    NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
    service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   16h

好的，經確認已經回復初始狀態

# 小結

今天我們看到了使用者的管理，以及基於RBAC的集群權限控管方式，最後我們看到了管理節點的基本工具，管理三劍客：Drain、Cordon、Uncordon，我們可以透過這些工具達成我們管理節點的需求。今天是管理篇就結束了，明天整合介紹一些Docker和k8s的常用套件，預計後天我們會開始介紹雲端的東西，明天見囉！

# 參考資料

- [k8s基於RBAC的用戶管理](https://medium.com/@HoussemDellai/rbac-with-kubernetes-in-minikube-4deed658ea7b)
- [k8s的RBAC授權](https://nicksors.cc/2019/07/09/kubernetes%E7%B3%BB%E5%88%97%E4%B9%8B%E3%80%8Ak8s%E5%9F%BA%E4%BA%8ERBAC%E7%9A%84%E6%8E%88%E6%9D%83%E3%80%8B.html)
- [k8s的授權](https://kubernetes.io/zh/docs/reference/access-authn-authz/authorization/)
- [k8s節點管理](https://k8smeetup.github.io/docs/tasks/administer-cluster/safely-drain-node/)
