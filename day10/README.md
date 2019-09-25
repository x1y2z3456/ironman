# 第十天：k8s之三：進階篇之一，Service Discovery、ConfigMap

Author: Nick Zhuang
Type: kubernetes

# 前言

昨天我們已經將基礎篇的部分做個結束，今天開始我們會介紹進階篇的內容，若是中途亂入的朋友，請務必複習前兩天的內容，再來看這邊的會比較清楚喔！今天會進一步介紹Service，以及ConfigMap的部分，那我們就開始吧～

# Service Discovery（服務發現）

同一個Pod裡面的container本來就可以互相溝通（localhost），這個部分是要介紹在不同的Pod裡面的container溝通的方式，可以透過Service Discovery這個機制去實現，我們這邊是要用DNS Pod與Service，藉以實現不同Pod之間的Container溝通。

這裡給的例子會複雜些，我們先編輯幾個檔案：

第一個是Secret，這裡儲存了用戶和密碼等資訊

    $vim secret.yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: helloworld-secrets
    type: Opaque
    data:
      username: aGVsbG93b3JsZA==
      password: cGFzc3dvcmQ=
      rootPassword: cm9vdHBhc3N3b3Jk
      database: aGVsbG93b3JsZA==

第二個是Database，這裡定義了與MySQL的連接

    $vim database.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: database
      labels:
        app: database
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        ports:
        - name: mysql-port
          containerPort: 3306
        env:
          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: helloworld-secrets
                key: rootPassword
          - name: MYSQL_USER
            valueFrom:
              secretKeyRef:
                name: helloworld-secrets
                key: username
          - name: MYSQL_PASSWORD
            valueFrom:
              secretKeyRef:
                name: helloworld-secrets
                key: password
          - name: MYSQL_DATABASE
            valueFrom:
              secretKeyRef:
                name: helloworld-secrets
                key: database

第三個是Database的Service，它讓Database這個Pod變得可見

    $vim database-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: database-service
    spec:
      ports:
      - port: 3306
        protocol: TCP
      selector:
        app: database
      type: NodePort

再來我們繼續看第四個，是helloworld-db的Deployment，它實踐了Pod的Horizontal Scaling，有3個ReplicaSet，也就是有3個副本可用，屆時Deployment會開啟3個Pods同時跑

    $vim helloworld-db.yaml
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: helloworld-deployment
    spec:
      replicas: 3
      template:
        metadata:
          labels:
            app: helloworld-db
        spec:
          containers:
          - name: k8s-demo
            image: 105552010/k8s-demo
            command: ["node", "index-db.js"]
            ports:
            - name: nodejs-port
              containerPort: 3000
            env:
              - name: MYSQL_HOST
                value: database-service
              - name: MYSQL_USER
                value: root
              - name: MYSQL_PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: helloworld-secrets
                    key: rootPassword
              - name: MYSQL_DATABASE
                valueFrom:
                  secretKeyRef:
                    name: helloworld-secrets
                    key: database

第五個是helloworld-db的Service，是為了要讓前者開起來的Pod可以被access

    $vim helloworld-db-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: helloworld-db-service
    spec:
      ports:
      - port: 3000
        protocol: TCP
      selector:
        app: helloworld-db
      type: NodePort

總就是5個檔案，再來我們把他們依序開起來

    $kubectl create -f secrets.yml
    secret/helloworld-secrets created
    $kubectl create -f database.yml
    pod/database created
    $kubectl create -f database-service.yml
    service/database-service created
    $kubectl create -f helloworld-db.yml
    deployment.extensions/helloworld-deployment created
    $kubectl create -f helloworld-db-service.yml
    service/helloworld-db-service created

我們看一下所有的元件狀態

    $kubectl get all
    NAME                                        READY   STATUS    RESTARTS   AGE
    pod/database                                1/1     Running   0          8m16s
    pod/helloworld-deployment-dd876ccc6-bj5ls   1/1     Running   0          38s
    pod/helloworld-deployment-dd876ccc6-cbfmn   1/1     Running   0          38s
    pod/helloworld-deployment-dd876ccc6-xbckk   1/1     Running   0          38s
    
    NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
    service/database-service        NodePort    10.96.173.28    <none>        3306:31964/TCP   8m7s
    service/helloworld-db-service   NodePort    10.101.99.153   <none>        3000:30283/TCP   21s
    service/kubernetes              ClusterIP   10.96.0.1       <none>        443/TCP          162d
    
    NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/helloworld-deployment   3/3     3            3           38s
    
    NAME                                              DESIRED   CURRENT   READY   AGE
    replicaset.apps/helloworld-deployment-dd876ccc6   3         3         3       38s

都確定設置好後，我們來看一下helloworld-db的Pod，檢查一下export的port

    $minikube service helloworld-db-service --url
    http://192.168.99.100:30283

接著我們來試著access這個Pod

    $curl http://192.168.99.100:30283
    Hello World! You are visitor number 1
    $curl http://192.168.99.100:30283
    Hello World! You are visitor number 2
    $curl http://192.168.99.100:30283
    Hello World! You are visitor number 3
    $curl http://192.168.99.100:30283
    Hello World! You are visitor number 4

因為有設定資料庫的關係，會自動加總access的次數，這是在index.js設定的

我們來看下MySQL資料庫的內容：

    $kubectl exec database -it -- mysql -u root -p
    Enter password:

這邊會要輸入密碼，這個密碼是在secret.yaml裡面定義的rootPassword，要如何把字串轉換回去，請參考昨天所介紹Secret的內容。

小提示：

    $echo "cm9vdHBhc3N3b3Jk" |base64 --decode
    #請自行DIY，這結果就是要輸入的密碼

成功進入後會有以下類似歡迎訊息：

    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 10
    Server version: 5.7.27 MySQL Community Server (GPL)
    
    Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.
    
    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.
    
    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

接著我們就進入MySQL的資料庫了！酷！我們來操作看看～

    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | helloworld         |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+
    5 rows in set (0.01 sec)

這邊提示了有5個MySQL的db可供選擇，這邊要看helloworld的內容

    mysql> use helloworld;
    Reading table information for completion of table and column names
    You can turn off this feature to get a quicker startup with -A
    
    Database changed

切換成觀察helloworld的資料庫成功！接著我們來看下這裡的資料表

    mysql> show tables;
    +----------------------+
    | Tables_in_helloworld |
    +----------------------+
    | visits               |
    +----------------------+
    1 row in set (0.00 sec)

可以看到有visits這個table，再來下SQL語法看它的內容

    mysql> select * from visits;
    +----+---------------+
    | id | ts            |
    +----+---------------+
    |  1 | 1568800680441 |
    |  2 | 1568800687965 |
    |  3 | 1568800689314 |
    |  4 | 1568800705884 |
    +----+---------------+
    4 rows in set (0.01 sec)
    
    mysql> \q
    Bye

可以看到有4筆資料，每個都有記錄time stamp

到這個地方，我們能瞭解到這個Pod及相關Service、Secret是如何運作的。

再來的部分就是，我們要檢查Pod以及Service的DNS，進而理解這樣的設置有達到Service Discovery的需求

我們先用kubectl檢查Service的訊息

    $kubectl get svc
    NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
    database-service        NodePort    10.96.173.28     <none>        3306:31964/TCP   31h
    helloworld-db-service   NodePort    10.110.204.153   <none>        3000:31264/TCP   23m
    kubernetes              ClusterIP   10.96.0.1        <none>        443/TCP          164d

我們運行一個BusyBox（輕量化的Linux系統鏡像），這個BusyBox帶有nslookup這個系統工具

    $kubectl run -i --tty busybox --image=busybox:1.28.4 --restart=Never -- sh
    If you don't see a command prompt, try pressing enter.
    / # nslookup
    BusyBox v1.28.4 (2018-05-22 17:00:17 UTC) multi-call binary.
    
    Usage: nslookup [HOST] [SERVER]
    
    Query the nameserver for the IP address of the given HOST
    optionally using a specified DNS server
    / # nslookup helloworld-db-service
    Server:    10.96.0.10
    Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
    
    Name:      helloworld-db-service
    Address 1: 10.110.204.153 helloworld-db-service.default.svc.cluster.local
    / # nslookup database-service
    Server:    10.96.0.10
    Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
    
    Name:      database-service
    Address 1: 10.96.173.28 database-service.default.svc.cluster.local

可以發現這裡顯示的IP跟用kubectl看到的相同

既然能確認DNS可以用，那麽它應該可以利用我們給它的名稱映射到IP

我們來驗證一下

    / # telnet helloworld-db-service 3000
    GET /
    
    HTTP/1.1 200 OK
    X-Powered-By: Express
    Content-Type: text/html; charset=utf-8
    Content-Length: 37
    ETag: W/"25-+yZ37Iv+ApgxmZ24TK89UA"
    Date: Wed, 18 Sep 2019 15:23:46 GMT
    Connection: close
    
    Hello World! You are visitor number 5Connection closed by foreign host

剛剛已經access了4次，這是第5次了，我們再進到MySQL資料庫看一次喔！

Ctrl+D跳出BusyBox

    $kubectl exec database -it -- mysql -u root -p
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | helloworld         |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+
    5 rows in set (0.01 sec)
    
    mysql> use helloworld;
    Reading table information for completion of table and column names
    You can turn off this feature to get a quicker startup with -A
    
    Database changed
    mysql> show tables;
    +----------------------+
    | Tables_in_helloworld |
    +----------------------+
    | visits               |
    +----------------------+
    1 row in set (0.00 sec)
    
    mysql> select * from visits;
    +----+---------------+
    | id | ts            |
    +----+---------------+
    |  1 | 1568800680441 |
    |  2 | 1568800687965 |
    |  3 | 1568800689314 |
    |  4 | 1568800705884 |
    |  5 | 1568820226530 |
    +----+---------------+
    5 rows in set (0.00 sec)
    
    mysql> exit
    Bye

可以看到有第5個time stamp！大功告成！

# ConfigMap

在設定檔的世界中，如果Secret代表神秘的月亮，記錄著機敏資料的話，那麼ConfigMap就代表閃亮的太陽，它是攤在陽光下的明碼設定，在k8s的集群當中，有些設定，是可以公開給人檢視和處裡的，這類型的範疇就可以使用ConfigMap去設定。

與Secret類似，ConfigMap也是key與value的pair。

我們來看幾個例子：

## 示例一：使用ConfigMap的Pod

第一個檔案：special-config.yaml

    $vim special-config.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: special-config
      namespace: default
    data:
      special.user: Nick
      special.role: Developer

第二個檔案：env-config.yaml

    $vim env-config.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: env-config
      namespace: default
    data:
      log_level: DEBUG

第三個檔案：用到special和env設定的Pod：pod-configmap.yaml

    $vim pod-configmap.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-test-configmap
    spec:
      containers:
        - name: test-container
          image: gcr.io/google_containers/busybox
          command: [ "/bin/sh", "-c", "echo $(SPECIAL_USER_KEY) is $(SPECIAL_ROLE_KEY)" ]
          env:
            - name: SPECIAL_USER_KEY
              valueFrom:
                configMapKeyRef:
                  name: special-config
                  key: special.user
            - name: SPECIAL_ROLE_KEY
              valueFrom:
                configMapKeyRef:
                  name: special-config
                  key: special.role
          envFrom:
            - configMapRef:
                name: env-config
      restartPolicy: Never

接著依序啟動

    $kubectl create -f special-config.yaml
    configmap/special-config created
    $kubectl create -f env-config.yaml
    configmap/env-config created
    $kubectl create -f pod-configmap.yaml
    pod/pod-test-configmap created

我們來看下集群內ConfigMap的狀態

    $kubectl get cm
    NAME             DATA   AGE
    env-config       1      17s
    special-config   2      23s

再來我們去看下對應的詳細內容：

special-config

    $kubectl describe cm special-config
    Name:         special-config
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>
    
    Data
    ====
    special.role:
    ----
    Developer
    special.user:
    ----
    Nick
    Events:  <none>

env-config

    $kubectl describe cm env-config
    Name:         env-config
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>
    
    Data
    ====
    log_level:
    ----
    DEBUG
    Events:  <none>

可以注意到都是明碼，被看光光啦！

來看一下Pod的日誌檔，正常的情況下他應該要輸出環境變數（env

    $kubectl logs pod-test-configmap
    Nick is Developer

可以發現環境變數有整合到句子中，看起來沒啥問題

## 示例二：使用ConfigMap＋Volume的Pod

我們再來做更細緻的操作，把Volume的元件加入剛才的設定

我們先修改pod-configmap.yaml的內容，更名為pod-configmap-volume.yaml

    $vim pod-configmap-volume.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-test-configmap-volume
    spec:
      containers:
        - name: test-container
          image: gcr.io/google_containers/busybox
          command: [ "/bin/sh","-c","cat /etc/config/special/special-key" ]
          volumeMounts:
          - name: config-volume
            mountPath: /etc/config
      volumes:
        - name: config-volume
          configMap:
            name: special-config
            items:
            - key: special.user
              path: special/special-key
      restartPolicy: Never

這個Pod會掛載Volume（裡面的參數是從ConfigMap來的），並將掛載的內容顯示出來

我們來看一下這個Pod日誌

    $kubectl logs test-pod-configmap-volume
    Nick

OK！將ConfigMap整合到Volume上也能用！讚！！

# 小結

我們今天第一個看到了Servcie進階的用途，它是可以透過DNS，藉以實現在群集中找到Pod的需求，理解了Service Discovery是確定如何連接服務的過程。為了驗證這個流程，我們先設計了DNS的Pod來搭配Service，並且透過MySQL資料庫去記錄被access的次數，接著透過BusyBox去檢查Service所顯示的IP，藉由nslookup去檢查DNS，最後透過發了一個HTTP的Request去驗證DNS的映射是沒有問題的。第二個部分是ConfigMap，我們定義了明碼的設定檔內容，這些內容是可以透過環境變數的方式在Pod內部使用。

### 小複習

還記得第七天提過的DevOps的12個要素嗎？這裡的元件有用到其中Config（配置）、Backing services（後端服務）、Processes（進程）、Port binding（端口綁定）、Disposability（易處理）、Logs（日誌）等等類似的觀念，稍微回顧一下，之前看到的這些觀念是不是變簡單了呢？

我們明天會延續今天的內容，繼續介紹進階篇的部分，明天見！

# 參考資料

- [Pod與DNS](https://kubernetes.io/zh/docs/concepts/services-networking/dns-pod-service/)
- [Pod與ConfigMap](https://k8smeetup.github.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
