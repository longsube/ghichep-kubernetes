# Hướng dẫn về việc sử dụng Storage trong K8S

## Mô hình lab:
 - Sử dụng 3 host cài đặt K8S: 172.16.68.83, 172.16.68.84, 172.16.68.85
 - Host 172.16.68.83 làm host master.


## 1. Các bước chuẩn bị
Đã triển khai cụm K8S theo hướng dẫn tại [đây](https://github.com/longsube/ghichep-kubernetes/blob/master/docs/CaiDatK8S.md)

## 2. Tạo PV (Persistent Volume)
### 2.1. Tạo file `persistentVolume.yaml` với nội dung
```sh
cat > persistentVolume.yaml << EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-local-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: my-local-storage
  local:
    path: /mnt/disk/vol1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - docker2
EOF
```

Kịch bản trên sẽ tạo ra 1 PV dung lượng 5GB, map với storage class là `my-local-storage`, sử dụng đường dẫn local là `/mnt/disk/vol1` trên host docker 2.

### 2.2. Trên host docker2, tạo thư mục local làm nơi để chứa PV của pod 
```sh
DIRNAME="vol1"
mkdir -p /mnt/disk/$DIRNAME 
chcon -Rt svirt_sandbox_file_t /mnt/disk/$DIRNAME
chmod 777 /mnt/disk/$DIRNAME
```

### 2.3. Trên host docker1 (master), chạy lệnh để khởi tạo PV
```sh
kubectl create -f persistentVolume.yaml
```
hoặc
```sh
kubectl apply -f persistentVolume.yaml
```

### 2.4. Kiểm tra
```sh
kubectl get pv  my-local-pv
```
Kết quả:
```sh
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS       REASON   AGE
my-local-pv   5Gi        RWO            Retain           Bound    default/my-claim   my-local-storage            16d
```

## 3. Tạo PVC (PersistentVolumeClaim)
### 3.1. Tạo file `persistentVolumeClaim.yaml` với nội dung
```sh
cat > persistentVolumeClaim.yaml << EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-claim
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: my-local-storage
  resources:
    requests:
      storage: 5Gi
EOF
```
Kịch bản trên sẽ tạo ra một Persistent Volume Claim có tên là `my-claim`, map với storage class là `my-local-storage`

### 3.2. Trên host docker1 (master), chạy lệnh để khởi tạo PVC
```sh
kubectl create -f persistentVolumeClaim.yaml
```
hoặc
```sh
kubectl apply -f persistentVolumeClaim.yaml
```

### 3.3. Kiểm tra
```sh
kubectl get pvc my-claim
```
Kết quả:
```sh
NAME       STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS       AGE
my-claim   Bound    my-local-pv   5Gi        RWO            my-local-storage   16d
```

## 4. Tạo Storage class
### 4.1. Tạo file `storageClass.yaml` với nội dung
```sh
cat > storageClass.yaml << EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: my-local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF
```

Kịch bản trên sẽ tạo ra 1 storage class với tên là `my-local-storage` sử dụng provisioner là storage local.

### 4.2. Trên host docker1 (master), chạy lệnh để khởi tạo Storage Class
```sh
kubectl create -f storageClass.yaml
```
hoặc
```sh
kubectl apply -f storageClass.yaml
```

### 4.3. Kiểm tra
```sh
kubectl get sc my-local-storage
```


## 5. Tạo pod để mount với PV
### 5.1. Tạo file `http-pod.yaml` với nội dung
```sh
cat > http-pod.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: www
  labels:
    name: www
spec:
  containers:
  - name: www
    image: nginx:alpine
    ports:
      - containerPort: 80
        name: www
    volumeMounts:
      - name: www-persistent-storage
        mountPath: /usr/share/nginx/html
  volumes:
    - name: www-persistent-storage
      persistentVolumeClaim:
        claimName: my-claim
EOF
```

### 5.2. Trên host docker1 (master), chạy lệnh để khởi tạo pod
```sh
kubectl create -f http-pod.yaml
```
hoặc
```sh
kubectl apply -f http-pod.yaml
```

### 5.3. Kiểm tra
```sh
kubectl get pod www
```
### 5.4. Kiểm tra việc mount volume, trên host docker2
```sh
echo "Hello local persistent volume" > /mnt/disk/vol1/index.html
```
Trên host docker1
```sh
POD_IP=$(kubectl get pod www -o yaml | grep podIP | awk '{print $2}'); echo $POD_IP
curl $POD_IP
```
Kết quả:
```sh
Hello local persistent volume
```

## Tham khảo:
- https://vocon-it.com/2018/12/20/kubernetes-local-persistent-volumes/
- https://kubernetes.io/docs/concepts/storage/volumes/#local