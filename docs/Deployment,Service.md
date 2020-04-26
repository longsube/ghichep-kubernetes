Tao Deployment
kubectl apply -f deploy.yml
kubectl get deploy hello-deploy
kubectl describe deploy hello-deploy
kubectl get rs

Tao Service
kubectl apply -f svc.yml

RollingUpdate
kubectl apply -f deploy.yml --record

RollBack
kubectl rollout history deployment hello-deploy
kubectl rollout undo deployment hello-deploy --to-revision=1
kubectl rollout status deployment hello-deploy

Imperative way
kubectl expose deployment web-deploy \
  --name=hello-svc2 \
  --target-port=8080 \
  --type=NodePort

kubectl get ep hello-svc