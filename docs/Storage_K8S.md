# Tổng hợp về việc sử dụng Storage trong K8S

### Tao PV

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

### Tren docker2
on the node, where the POD will be located (node1 in our case):
DIRNAME="vol1"
mkdir -p /mnt/disk/$DIRNAME 
chcon -Rt svirt_sandbox_file_t /mnt/disk/$DIRNAME
chmod 777 /mnt/disk/$DIRNAME

### Tren docker1 (master)
kubectl create -f persistentVolume.yaml
hoac
kubectl apply -f persistentVolume.yaml

Kiem tra
kubectl get pv  my-local-pv

### Tao PVC
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

kubectl create -f persistentVolumeClaim.yaml
hoac
kubectl apply -f persistentVolumeClaim.yaml

Kiem tra
kubectl get pvc my-claim

### Tao pod de mount voi PV
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

kubectl create -f http-pod.yaml
hoac
kubectl apply -f http-pod.yaml

Kiem tra
kubectl get pod www

### Tao Storage class
cat > storageClass.yaml << EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: my-local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF

kubectl create -f storageClass.yaml
hoac
kubectl apply -f storageClass.yaml

Kiem tra
kubectl get sc my-local-storage

### Kiem tra viec mount volume
Tren docker2
echo "Hello local persistent volume" > /mnt/disk/vol1/index.html

Tren docker1
POD_IP=$(kubectl get pod www -o yaml | grep podIP | awk '{print $2}'); echo $POD_IP
curl $POD_IP
Hello local persistent volume

Tham khao:
- https://vocon-it.com/2018/12/20/kubernetes-local-persistent-volumes/
- https://kubernetes.io/docs/concepts/storage/volumes/#local