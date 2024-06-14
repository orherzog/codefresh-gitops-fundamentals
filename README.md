# codefresh-gitops-fundamentals

Notes from Codefresh GitOps Fundamentals Course Practice

## Table of Contents

- [ArgoCD installation on Minikube](#argocd-installation-on-minikube)

## ArgoCD intallation on minikube

### Manual installation 
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
### Installation using Helm
```bash
kubectl create namespace argocd
git clone https://github.com/argoproj/argo-helm.git
cd argo-helm/charts/argo-cd
helm repo add dandydeveloper https://dandydeveloper.github.io/charts/
helm repo update
helm dependency build
helm install argocd . --namespace argocd
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Expose Argo using port-forward
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Expose Argo using service with Nodeport
Apply the following service manifest to expose the 8080 container port to 30443 to the node:
```bash
apiVersion: v1
kind: Service
metadata:
  labels:
    app: argocd-server
  managedFields:
  name: argocd-server-nodeport
  namespace: argocd
spec:
  ports:
  - nodePort: 30443
    port: 8080
    protocol: TCP
  selector:
    app.kubernetes.io/name: argocd-server
  type: NodePort
```

## Use local helm chart directory
```bash 
mkdir argocd
cd argocd
cp -r ../argo-helm/charts/argo-cd/* .
cd ..
rm -rf argo-helm
```

## Install ArgoCD CLI
```bash 
# On macOS
brew install argocd

# On Linux
sudo curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd
```

## Login to ArgoCD using admin user
```bash 
argocd login localhost:8080 --insecure
```
- Username: admin
- Password: G5dm2hKkwTqOLC0t (Retrive from the kubernets secret that was created using the installation)
```bash 
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Creating an Argo CD application with the argocd CLI

- Create an .env file, in your project directory. This file will contain key-value pairs of your environment variables:

```bash
# .env file
APP_NAME=my-app
PROJECT=default
GIT_REPO=https://github.com/orherzog/codefresh-gitops-fundamentals.git
APP_FOLDER=manifests
NAMESPACE=default
SERVER_URL=https://kubernetes.default.svc
```
- Create ArgoCD application:

```bash
argocd app create $APP_NAME \
--project $PROJECT \
--repo $GIT_REPO \
--path $APP_FOLDER \
--dest-namespace $NAMESPACE \
--dest-server $SERVER_URL
```