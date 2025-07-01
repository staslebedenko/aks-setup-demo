# Step 1. Introduction, Apps, Kubernetes and Argo CD

- Intro - what GitOps is and isn’t
- Workshop structure and goals
    - Try to do as much as possible during workshop and continue home, because content can be bigger than 2h
    - I will be going through workshop steps explaining them
- Practicalities
    - Application code, manifests, public registry
    - Checking if local tools are installed - docker, kubectl, kustomize ?
    - Connecting to Azure and AKS with --admin option
    - Installing Argo
    - Init of infra repository
    - Port forwarding on different command lines
    - Disabling Argo default project
    - Adding new user to Argo

## Applications
Applications prepared as two custom containers plus Redis instance, containers already deployed to public Docker registry and referenced in yaml manifests.
Plese do not deploy manifests with Kubectl, we will do this via Argo CD later.

I explained manifests and docker-compose setup at the end of this readmy, if case you will need it :).

## Argo CD setup

Login to Azure with device code from CMD
```yaml
az login --use-device-code
```

Deploy AKS cluster
```yaml
az group create --name devbcn-demo --location westeurope
az aks create --name devbcn-cluster --resource-group devbcn-demo --location westeurope --node-resource-group devbcn-demo-resources --enable-managed-identity --node-count 2 --generate-ssh-keys --enable-oidc-issuer --enable-workload-identity  
```

After cluster deployment we need to connect to it and add to local kube context

Login to Azure AKS access token and switch context to the current cluster
```yaml
az aks get-credentials --resource-group devbcn-demo --name devbcn-cluster --admin
kubectl config use-context devbcn-cluster
kubectl get all
```

The next step would be setup default instance of Argo CD from public manifest 
```yaml
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get all -n argocd
```

Next we need to output admin password for the instance we just deployed 

For Powershell terminal (I'm using one in Visual Studio Code)
```yaml
$encodedPass = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"  
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encodedPass))  
```

(Optional)Getting password in CMD terminal
```yaml
for /f %i in ('kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath^="{.data.password}"') do @powershell "[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String(\"%i\"))"  
```

Please record your password somewhere safe, it looks something like below :)
C6#453$#532432E

Start background port forwarding and login to our Argo CD instance for CMD
```yaml
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
```

(Important)Don’t forget to kill background kubectl tasks :), works both for PS and CMD
```yaml
taskkill /IM kubectl.exe /F
```

Alternative PowerShell terminal command for port-forwarding
```yaml
$job = Start-Job -ScriptBlock { kubectl port-forward svc/argocd-server -n argocd 8080:443 }
```
And killing of the process
```yaml
Stop-Job $job  
Remove-Job $job
```

Visit [https://localhost:8080](https://localhost:8080/)  Argo CD URL and login

Also make sure that you can login with Argo CD CLI, because this will be needed at the next step

```yaml
argocd logout localhost:8080 
argocd login localhost:8080 --username admin --password C6#453$#532432E  --insecure
argocd app list  
```

Repeating kubectl cleanup command
```yaml
taskkill /IM kubectl.exe /F
```


## Application extras
If you need to test local applications please use following code from applications folder
```yaml
docker-compose up --build
docker-compose down
docker-compose up --build
```

If you need to rebuild apps and you are using Docker, then adjust remote repository name below
```yaml
docker login

docker build -t stasiko/funneverends-frontend:latest -f frontend/Dockerfile ./frontend
docker push stasiko/funneverends-frontend:latest

docker build -t stasiko/funneverends-backend:latest -f backend/Dockerfile ./backend
docker push stasiko/funneverends-backend:latest
```

Login to Azure AKS access token and switch context to the current cluster
```yaml
az login --use-device-code
az aks get-credentials --resource-group devbcn-demo --name devbcn-cluster --admin
kubectl config use-context devbcn-cluster
```

If you want to just deploye apps to empty kubernetes cluster, you can use following order of commands
```yaml
kubectl apply -f namespace.yaml
kubectl apply -f backend-configmap.yaml
kubectl apply -f backend-secret.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f redis-deployment.yaml
```

Check deployed services with following commands
```yaml
kubectl get configmaps,svc -n devbcn
kubectl get configmaps -n guestbook
kubectl get svc frontend 
kubectl get svc backend
```

if you need to delete namespace after manual deployment and testing
```yaml
kubectl delete namespace devbcn
```

To access and check if applications working in the remote claster, please start port forwarding as background tasks
```yaml
kubectl port-forward svc/backend -n devbcn 5000:5000 &
kubectl port-forward svc/frontend -n devbcn 8080:80 &
kubectl port-forward svc/redis -n devbcn 6379:6379 &
```

And don’t forget to kill kubectl processes afterwards with
```yaml
taskkill /IM kubectl.exe /F
```
