## Deploy a basic nginx application(HTML) to aks using argo cd
0.	use should have a github repo like below with the structure and content inside it

```bash
LakshmiNarayana@argo-vm:~/lucky-bakery-argocd$ tree
.
├── app
│   ├── Dockerfile
│   └── index.html
└── k8s
    ├── application.yaml
    ├── deployment.yaml
    └── service.yaml
```
1. Deploy a Azure VM with at least 2 GB of Ram and all ports open
2. Install Nginx
```bash
sudo apt  update
sudo apt install nginx -y
sudo systemctl start nginx
```
4. Install Az Cli and login to your Account
```
sudo apt update
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az login –use-device-code
```
5. Create resource group and aks cluster
```
az group create --name rg-argo --location eastus
az aks create   --resource-group rg-argo   --name lucky-aks-cluster   --node-count 1   --generate-ssh-keys
az aks get-credentials --resource-group rg-argo --name lucky-aks-cluster
```
6. Install Git
```bash
sudo apt update
sudo apt install git -y
```  
7. Install Kubectl inside the VM
```bash
sudo apt update
sudo snap install kubectl --classic
```
8. Install docker inside the VM
```bash
sudo apt update -y
wget -O docker.sh https://get.docker.com/
sudo sh docker.sh
sudo groupadd -f docker
sudo usermod -aG docker "$USER"
sudo systemctl restart docker
newgrp docker
docker --version
docker ps
```

9. Install ArgoCD inside Kubernetes and accessing it
  - ArgoCD: Argo CD is 100% Kubernetes-native. It is built specifically to deploy and manage applications inside Kubernetes clusters.
  - Think of Argo CD as: A GitOps controller that runs inside a Kubernetes cluster and continuously ensures the cluster matches the configuration stored in Git.

- Create a namespace in Kubernetes
```bash
kubectl create namespace argocd
kubectl get namespaces   #you will see argocd namespace created
```
- A namespace in Kubernetes is just a logical partition inside a cluster.
- Think of it as a folder inside your Kubernetes cluster where you keep related resources together.
- Paste the Official repo
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml 
kubectl get svc -n argocd #you will see argocd services
```
- By default, the Argo CD API server is not exposed with an external IP. To access the API server do below
- This below command exposes the ArgoCD server to the internet by converting it into a LoadBalancer service and giving it a public IP.
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
- Copy the public IP of argocd-server using below
```bash
kubectl get svc -n argocd # you will see all services in argocd namespace
```
- For password of argocd server use below
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
- make a note of that password
- Browse the public ip which you copied with 443 as it is not https it show error but click on continue to site and navgate to argoCD UI
  - Username : admin
  - Password : which you copied from above command 
10. Build & Push Docker Image to Docker Hub
1.1 Log in to Docker Hub on your VM
- On your VM:
```bash
docker login
```
- Clone the repo where all your code resides to the VM
```bash
git clone https://github.com/luckysuie/lucky-bakery-argocd 
cd ~/lucky-bakery-argocd/app
docker build -t <dockerhub-username>/lucky-bakery:v1 .
docker push <dockerhub-username>/lucky-bakery:v1
```
- Connect / Sync with Argo CD
  - Using application.yaml
  - Apply the Argo CD Application from your VM:
```bash
kubectl apply -n argocd -f k8s/application.yaml
```
- Check in Argo CD:
```bash
kubectl get applications -n argocd
kubectl get svc lucky-bakery-service -n default
```
- browse the external  public ip after performing the above command
    - you should see your application

<img width="1879" height="966" alt="Screenshot 2025-11-29 165645" src="https://github.com/user-attachments/assets/7d842bda-076b-4e36-8340-3db3f7e87b2e" />

## Now test GitOps behaviour (this is what you want)
- Test 1 – Change in Git → Argo CD updates AKS
- Goal: Show that changing YAML in Git changes the cluster automatically.
1.	Edit k8s/deployment.yaml in your repo and change replicas:
```bash
spec:
  replicas: 3
```
2.	Commit and push:
```bash
git add k8s/deployment.yaml
git commit -m "Scale Lucky Bakery to 3 replicas"
git push origin main
```
3.	Go to Argo CD UI:  You should see lucky-bakery become OutOfSync → Syncing → Healthy.
4.	Check pods:
```bash
kubectl get pods -n default -l app=lucky-bakery
```
- You should now see 3 pods.
- Explanation to trainees:
  - We did not run kubectl apply. We only changed the file in Git. Argo CD pulled the change and updated the cluster.

### Argo CD UI with Pods
<img width="1893" height="942" alt="Screenshot 2025-11-29 164308" src="https://github.com/user-attachments/assets/f25f9d3c-a066-4c40-806d-e977130d2491" />


