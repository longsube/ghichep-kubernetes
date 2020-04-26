# Hướng dẫn cài đặt K8S sử dụng kubeadm.

## Mô hình lab:
 - Sử dụng 3 host cài đặt K8S: 172.16.68.83, 172.16.68.84, 172.16.68.85
 - Host 172.16.68.83 làm host master.


## 1. Các bước chuẩn bị
### 1.1. Update và lấy key
```sh
apt-get update
```

```sh
apt-get install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  software-properties-common
```

```sh
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
Kết quả:
```sh
OK
```

```sh
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
Kết quả:
```sh
OK
```

### 1.2. Cấu hình repo để cài đặt các package của K8S. Sửa file `vim /etc/apt/sources.list.d/kubernetes.list` 
```sh
vim /etc/apt/sources.list.d/kubernetes.list
```
Thêm vào:
```sh
deb https://apt.kubernetes.io/ kubernetes-xenial main
```

### 1.3. Cài đặt docker container trên tất cả các host
```sh
apt-get update
apt-get install -y kubelet kubeadm kubectl
sudo apt-key fingerprint
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
apt-get update
apt-get install docker-ce
```

### 1.4. Khai báo `/etc/hosts` trên tất cả các host
```sh
172.16.68.83    docker1
172.16.68.84    docker2
172.16.68.85    docker3
```

## 2.  Triển khai cụm K8S
### 2.1. Trên host master, khởi tạo cụm K8S
```sh
kubeadm init
```

Kết quả:
```sh
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.16.68.83:6443 --token vyp531.zqlaby0861a9qubr \
    --discovery-token-ca-cert-hash sha256:ba6efecb4780d25d5241a8ba0e1c7ff92a8cce609baa747da68770010e336001 
```

### 2.2. Thực hiện lệnh xong để copy các file config từ `/etc/kubernetes` vào thư mục của user và change quyền
```sh
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 2.3. Verify lại việc cài đặt
```sh
kubectl get nodes
```
Kết quả:
```sh
NAME      STATUS     ROLES    AGE   VERSION
docker1   NotReady   master   9h    v1.18.0
```
### 2.4. Kiểm tra các pod đã được tạo trong quá trình cài đặt
```sh
kubectl get pods --all-namespaces
```
Kết quả:
```sh
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
kube-system   coredns-66bff467f8-q7x45          0/1     Pending   0          9h
kube-system   coredns-66bff467f8-v2cbq          0/1     Pending   0          9h
kube-system   etcd-docker1                      1/1     Running   0          9h
kube-system   kube-apiserver-docker1            1/1     Running   0          9h
kube-system   kube-controller-manager-docker1   1/1     Running   0          9h
kube-system   kube-proxy-6fplg                  1/1     Running   0          9h
kube-system   kube-scheduler-docker1            1/1     Running   0          9h
```
Ta có thể thấy các pod coredns đang Pending, chờ 1 lúc để các pod này chuyển trạng thái về Running.

### 2.5. Tạo pod network, trong bài lab này sử dụng Weave
```sh
kubectl apply -n kube-system -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
Kết quả:
```sh
serviceaccount/weave-net created
```

### 2.6. Add các host 2 và 3 làm worker node.
Trên host 2, chạy lệnh sau với token và cert token là kết quả khi tạo K8S cluster. Lưu ý: token chỉ tồn tại trong 24h, để lấy token mới, sử dụng lệnh `kubeadm token create --print-join-command`
```sh
kubeadm join 172.16.68.83:6443 --token vyp531.zqlaby0861a9qubr \
    --discovery-token-ca-cert-hash sha256:ba6efecb4780d25d5241a8ba0e1c7ff92a8cce609baa747da68770010e336001 
```

### 2.7. Tương tự như host 1, kiểm tra các pod tới khi Running hoàn toàn
```sh	
kubectl get pods --all-namespaces
```

## 3. Thử nghiệm
### 3.1. Tạo 1 file pod.yml với nội dung
```sh
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
  labels:
    zone: prod
    version: v1
spec:
  containers:
  - name: hello-ctr
    image: nigelpoulton/k8sbook:latest
    ports:
    - containerPort: 8080
```
### 3.2. Tạo pod với name là hello-pod, public port 8080
```sh
kubectl apply -f pod.yml
```
### 3.3. Kiểm tra
```sh
kubectl get pods hello-pod
```
Kết quả:
```sh
NAME   READY   STATUS    RESTARTS   AGE
hello-pod    1/1     Running   0          14d
```
Có thể xem nhiều thông tin về pod hơn bằng lệnh:
```sh
kubectl get pods hello-pod -o yaml
```
### 3.4. Để xem thông tin của các container của pod
```sh
kubectl describe pods hello-pod
```
### 3.5. Để truy cập vào shell của container trong pod
```sh
kubectl exec -it hello-pod --container hello-ctr sh
```
hello-ctr là tên container, nếu ko có sẽ mặc định vào container đầu tiên trong pod.
### 3.6. Để lấy log của container trong pod
```sh
kubectl logs hello-pod --container hello-ctr
```
### 3.7. Xóa pod
```sh
kubectl delete -f pod.yml
```

## Tham khảo:
- Kubernetes Book: https://www.amazon.com/Kubernetes-Book-Version-November-2018-ebook/dp/B072TS9ZQZ/ref=sr_1_2?dchild=1&keywords=Kubernetes+Book&qid=1587887630&sr=8-2