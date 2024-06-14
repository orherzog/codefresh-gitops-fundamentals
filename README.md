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
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
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
nano .env

# .env file
APP_NAME=nginx
PROJECT=default
GIT_REPO=https://github.com/orherzog/codefresh-gitops-fundamentals.git
APP_FOLDER=manifests
NAMESPACE=default
SERVER_URL=https://kubernetes.default.svc
```

- Apply the .env file
```bash
source .env
```

- Add remote repository to ArgoCD
```bash
argocd repo add https://github.com/orherzog/codefresh-gitops-fundamentals.git --username "myusername" --password "git-token"
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

## Verify that the application was installed successfully

```bash
argocd app get nginx
```

Expected results:
```bash
Name:               argocd/nginx
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://argocd.example.com/applications/nginx
Source:
- Repo:             https://github.com/orherzog/codefresh-gitops-fundamentals.git
  Target:           
  Path:             manifests
SyncWindow:         Sync Allowed
Sync Policy:        Manual
Sync Status:        Synced to  (76f37aa)
Health Status:      Healthy

GROUP  KIND        NAMESPACE  NAME              STATUS  HEALTH   HOOK  MESSAGE
       Service     default    nginx-service     Synced  Healthy        service/nginx-service created
apps   Deployment  default    nginx-deployment  Synced  Healthy        deployment.apps/nginx-deployment unchanged
```

## Synchronizing an Argo CD application

```bash
argocd app sync nginx
```

This synchronizes the application. To confirm it’s running, you can execute a kubectl command:
```bash
kubectl get all
```

* The application will have a status “Running” if synchronized successfully.

## Argo CD auto-sync

Argo CD will auto-sync applications on its own (if you enable it).

By default, the sync period is 3 minutes. This means that every 3 minutes Argo CD will:

* Gather all applications that are marked to auto-sync.
* Fetch the latest Git state from the respective repositories.
* Compare the Git state with the cluster state.
* Take an action:
    - If both states are the same, do nothing and mark the application as synced.
    - If states are different mark the application as "out-of-sync".

You can change the default period by editing the argocd-cm configmap found on the same namespace as Argo CD.

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  timeout.reconciliation: 240s # 4 Minutes
```
After you change the manifest, you need to redeploy the argocd-repo-server service.

* You can disable the sync functionality completely by setting the timeout.reconciliation period to 0.

## Git Webhook to trigger Argo sync

Argo CD can learn about Git change also using Git webhooks. Using Webhooks is very efficient, as now your Argo CD installation will never delay when you commit something to Git. If you only use the default way of polling, then you might have to wait up to 3 minutes for Argo CD to detect the changes. With Webhooks, as soon as there is any change in Git, Argo CD will run the sync process.

## Application Health

Apart from the synced/out-of-sync status, Argo CD also keeps track of the service health for each deployed application.

The decision whether an application is healthy or not depends on the type of underlying Kubernetes resources.

For the built-in Kubernetes resources, the rules are the following:

* Deployments, ReplicaSets, StatefulSets, and Daemon sets are considered “healthy” if observed generation is equal to desired generation.
* Number of updated replicas equals the number of desired replicas.

For a service of type Loadbalancer or Ingress, the resource is healthy if the status.loadBalancer.ingress list is non-empty, with at least one value for hostname or IP.

For custom Kubernetes resources, health is defined in Lua scripts. You can add your own health check for a custom resource by implementing the check in Lua which is a self-contained scripting language. You can write your own health check in a similar manner for your custom resource, if it is not already supported by Argo CD.

Note that having a custom health check just provides a better experience with Argo CD. You can still deploy your custom application even without a health check, and it will never show as Healthy in the Argo CD dashboards.

Argo CD health checks are completely independent from Kubernetes health probes.

## Manual or automatic sync Strategies

Manual or automatic sync defines what Argo CD does when it finds a new version of your application in Git. 
 * Automatic - If set to automatic, Argo CD will apply the changes then update/create new resources in the cluster. 
 * Manual - If set to manual, Argo CD will detect the change but will not change anything in the cluster.

## Auto-pruning
Auto-pruning defines what Argo CD does when you remove/delete files from Git. 
 * Enabled - If it is enabled, Argo CD will also remove the respective resources in the cluster as well. 
 * Disabled - If disabled, Argo CD will never delete anything from the cluster.

## Self-heal
Self-heal defines what Argo CD does when you make changes directly to the cluster (via kubectl or any other way). 
 * If enabled, then Argo CD will discard the extra changes and bring the cluster back to the state described in Git.

 ## Managing Secrets

Argo CD can be used with any secret solution that you already have deployed.

All solutions that handle secrets using Git are storing them in an encrypted form. This means that you can get the best of both worlds. Secrets can be managed with GitOps, and they can also be placed in a secure manner in any Git repository (even public repositories).