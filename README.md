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

## Login to ArgoCD 
```bash 
argocd login localhost:8080
```
- Username: admin
- Password: G5dm2hKkwTqOLC0t (Retrive from the kubernets secret that was created using the installation)
```bash 
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```