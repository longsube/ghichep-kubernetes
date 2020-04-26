# Hướng dẫn tạo Deployment và Service.

## Mô hình lab:
 - Sử dụng 3 host cài đặt K8S: 172.16.68.83, 172.16.68.84, 172.16.68.85
 - Host 172.16.68.83 làm host master.


## 1. Các bước chuẩn bị
Đã triển khai cụm K8S theo hướng dẫn tại ![đây](https://github.com/longsube/ghichep-kubernetes/blob/master/docs/CaiDatK8S.md)


## 2. Tạo Deployment

### 2.1. Tạo file `deploy.yml` với nội dung
```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hello-world
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-pod
        image: nigelpoulton/k8sbook:latest
        ports:
        - containerPort: 8080
   ```
### 2.2. Tạo deployment với name là `hello-deploy`
```sh  
kubectl apply -f deploy.yml
```
### 2.3. Để kiểm tra trạng thái của Deployment vừa tạo
```sh
kubectl get deploy hello-deploy
```
Kết quả:
```sh
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
hello-deploy   10/10   10           10          17d
```
### 2.4. Để xem chi tiết thông tin của Deployment, dùng lệnh
```sh
kubectl describe deploy hello-deploy
```

### 2.5. Deployment sẽ tự động triển khai ReplicaSet, kiểm tra bằng lệnh
```sh
kubectl get rs
```
Kết quả:
```sh
NAME                      DESIRED   CURRENT   READY   AGE
hello-deploy-8d494c7f6    10        10        10      17d
```

## 3. RollingUpdate
Trong `deploy.yml`, việc update được khai báo sử dụng RollingUpdate strategy:
 - `maxUnavailable: 1`: trong quá trình update, deployment sẽ không bao giờ thấp hơn 1 pod so với số pod quy định (9)
 - `maxSurge: 1`: trong quá trình update, deployment sẽ không bao giờ vượt quá 1 pod so với số pod quy định là (11)

### 3.1. Thay đổi nội dung của file `deploy.yml`, đổi `nigelpoulton/k8sbook:latest` -> `nigelpoulton/k8sbook:edge`
```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hello-world
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-pod
        image: nigelpoulton/k8sbook:edge
        ports:
        - containerPort: 8080
```

### 3.2. Update deployment mới
```sh
kubectl apply -f deploy.yml --record
```
option `--record` để lưu lại log thay đổi, log này có thể được xem lại bằng lệnh `kubectl rollout history deployment hello-deploy`

### 3.3. Giám sát việc update
```sh
kubectl rollout status deployment hello-deploy
```

### 3.4. Dùng lệnh sau để liệt kê ra các revision, là các history của mỗi update
```sh
kubectl rollout history deployment hello-deploy
```

### 3.5. Rollback lại một revision trước đó
```sh
kubectl rollout undo deployment hello-deploy --to-revision=1
```

### 3.6. Để xóa 1 deployment
```sh
kubectl delete -f deploy.yml
```

Service có thể được tạo theo Declative way hoặc Imperative way
 - Declative way: khai báo các chỉ dẫn dưới dạng file service.yml (bao gồm label của deployment), cho phép tạo Service mới trên các pod và Deployment đang chạy
 - Imperative way: tạo Service ngay trên các Deployment đang chạy mà không đưa ra các chỉ dẫn, cách này giúp việc tạo service nhanh nhưng khó maintain về sau.
Trong thực tế sẽ triển khai service theo Declative way là chủ yếu.

## 4. Tạo Service theo Declative way

### 4.1. Tạo file `svc.yml` với nội dung
```sh
apiVersion: v1
kind: Service
metadata:
  name: hello-svc
  labels:
    app: hello-world
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 30001
    protocol: TCP
  selector:
    app: hello-world
```

### 4.2. Tạo Service với name hello-svc cho các pod có label `hello-world`
```sh
kubectl apply -f svc.yml
```
### 4.3. Kiểm tra
```sh
kubectl get svc hello-svc
```
Kết quả:
```sh
NAME        TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
hello-svc   NodePort   10.110.213.213   <none>        8080:30001/TCP   17d
```

### 4.4. Để xem thông tin chi tiết hơn về service
```sh
kubectl describe svc hello-svc
```
Kết quả:
```sh
Name:                     hello-svc
Namespace:                default
Labels:                   app=hello-world
Annotations:              Selector:  app=hello-world
Type:                     NodePort
IP:                       10.110.213.213
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30001/TCP
Endpoints:                10.36.0.1:8080,10.36.0.10:8080,10.36.0.2:8080 + 17 more...
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

### 4.5. Liệt kê các endpoint của service
```sh
kubectl get ep hello-svc
```
Kết quả:
```sh
NAME        ENDPOINTS                                                    AGE
hello-svc   10.36.0.1:8080,10.36.0.10:8080,10.36.0.2:8080 + 17 more...   17d
```

## 5. Tạo Service theo Imperative way

### 5.1. Tạo file `web-deploy.yml` với nội dung
```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-ctr
        image: nigelpoulton/k8sbook:latest
        ports:
        - containerPort: 8080
```

### 5.2. Tạo Deployment với name là web-deploy
```sh
kubectl apply -f web-deploy.yml 
```

### 5.3. Tạo service cho Deployment web-deploy vừa tạo
```sh
kubectl expose deployment web-deploy \
  --name=hello-svc2 \
  --target-port=8080 \
  --type=NodePort
```

### 5.4. Kiểm tra 
```sh
kubectl get svc hello-svc2
```


## Tham khảo:
- Kubernetes Book: https://www.amazon.com/Kubernetes-Book-Version-November-2018-ebook/dp/B072TS9ZQZ/ref=sr_1_2?dchild=1&keywords=Kubernetes+Book&qid=1587887630&sr=8-2