# Use Network and GPU Operator for GPUDirect RDMA in AKS cluster
This document inlcudes steps to apply Nvidia Network Operator for MOFED driver and Nvidia GPU Operator for GPU driver in AKS GPU nodepool for GPUDirect RDMA.

## Installation
### Prerequisites
- Azure subscription with GPU quota(example is using NDA100v4)
- A Client with Azure cli installed
- WSL for Windows

### Deploy Azure resources
Install kubectl:
```
az extension add --name aks-preview
az aks install-cli
```

Deploy AKS cluster and add GPU node pool:
```
az aks create -g ResourceGroupName -n AKSClusterName --ssh-key-value SSHKEYPATH --location REGION
az aks get-credentials --resource-group ResourceGroupName --name AKSClusterName
az aks nodepool add --resource-group ResourceGroupName --cluster-name AKSClusterName --name GPUNodePoolName --node-count 1 --node-vm-size Standard_ND96asr_v4  --tags SkipGPUDriverInstall=True --enable-node-public-ip --node-public-ip-prefix ResourceIdOfPublicIpPrefix
az aks get-credentials --resource-group ResourceGroupName --name myAKSCluster
```
![image](https://github.com/user-attachments/assets/b602539e-1fc7-48b7-a9f1-5011c702c8fb)

Note: Enable gpu node public ip is for debugging purpose. If there is pod failure, we can ssh into the node to check pod log.

### Install Helm in WSL:
```
# Make sure the kube config file is in the correct path:
cp /mnt/c/Users/USERNAME/.kube/config ${HOME}/.kube/
wget https://get.helm.sh/helm-v3.17.2-linux-amd64.tar.gz
tar -zxvf helm-v3.0.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo add nvidia https://nvidia.github.io/gpu-operator
helm repo update
```
### Install Nvidia Network Operator and GPU Operator
The NVIDIA GPPU and Network operators are used to manage the GPU drivers and Infiniband drivers on the NDv5 nodes. 
```
kubectl create namespace nvidia-operator

# Install node feature discovery
helm upgrade -i --wait \
  -n nvidia-operator node-feature-discovery node-feature-discovery \
  --repo https://kubernetes-sigs.github.io/node-feature-discovery/charts \
  --set-json master.nodeSelector='{"kubernetes.azure.com/mode": "system"}' \
  --set-json worker.nodeSelector='{"kubernetes.azure.com/accelerator": "nvidia"}' \
  --set-json worker.config.sources.pci.deviceClassWhitelist='["02","03","0200","0207"]' \
  --set-json worker.config.sources.pci.deviceLabelFields='["vendor"]'

# Install the network-operator
helm upgrade -i --wait \
  -n nvidia-operator network-operator network-operator \
  --repo https://helm.ngc.nvidia.com/nvidia \
  --set deployCR=true \
  --set nfd.enabled=false \
  --set ofedDriver.deploy=true \
  --set secondaryNetwork.deploy=false \
  --set rdmaSharedDevicePlugin.deploy=true \
  --set sriovDevicePlugin.deploy=true \
  --set-json sriovDevicePlugin.resources='[{"name":"mlnxnics","linkTypes": ["infiniband"], "vendors":["15b3"]}]'
# Note: use --set ofedDriver.version="<MOFED VERSION>"
#       to install a specific MOFED version
#
# Install the gpu-operator
helm upgrade -i --wait \
  -n nvidia-operator gpu-operator gpu-operator \
  --repo https://helm.ngc.nvidia.com/nvidia \
  --set nfd.enabled=false \
  --set driver.enabled=true \
  --set driver.version="535.230.02" \
  --set driver.rdma.enabled=true \
  --set toolkit.enabled=true
Note: When choosing the GPU driver ersion, you need to make sure that the it applies for the VM SKU you are choosing. You can confirm that by deploying a standalone VM and installing the driver.
```
Apply nicclusterpolicy to download and install MOFED driver:
```
wget https://raw.githubusercontent.com/Mellanox/network-operator/refs/heads/master/example/crs/mellanox.com_v1alpha1_nicclusterpolicy_cr.yaml
sed -i 's|repository: nvcr.io/nvstaging/mellanox|repository: nvcr.io/nvidia/mellanox|' mellanox.com_v1alpha1_nicclusterpolicy_cr.yaml
sed -i 's|version: 25.04-0.0.6.0-0|version: 24.10-0.7.0.0-0|' mellanox.com_v1alpha1_nicclusterpolicy_cr.yaml
# This is to pull ofed driver 24.10-0.7.0.0-0 image from repository nvcr.io/nvidia/Mellanox.

kubectl apply -f mellanox.com_v1alpha1_nicclusterpolicy_cr.yaml
```
Wait for sometime for those pods to be up and running. If any errors, you can check the pod logs under /var/logs/pod/xxx in node level.

Check pod status:
```
kubectl get pods -n nvidia-operator
```
![image](https://github.com/user-attachments/assets/4c8493be-e2cd-48aa-a79d-7efaafc65f3a)


### Run Node Health Check
```
az login 
az acr create    --resource-group ResourceGroupName    --name $ACR_NAME  --sku Basic    --admin-enabled
az acr login -n $ACR_NAME
az acr credential show --name $ACR_NAME
kubectl create secret docker-registry acr-auth   --docker-server=$ACR_NAME.azurecr.io  --docker-username=$ACR_NAME  --docker-password=xxx
git clone https://github.com/edwardsp/ai-on-aks.git
cd ai-on-aks
helm install health-check ./examples/healthcheck --set numNodes=1
cd docker/aksnhc

docker build -t $ACR_NAME.azurecr.io/aksnhc .
docker push $ACR_NAME.azurecr.io/aksnhc

kubectl -n nvidia-operator get pods

dos2unix nccl-tests.sh
cd docker/nccl
docker build -t elsatestacr.azurecr.io/nccltest .
docker push elsatestacr.azurecr.io/nccltest
```

### Additional info for troublehsooting purpose:
```
Remove gpu-operatore steps(run in WSL):
helm delete -n nvidia-operator $(helm list -n nvidia-operator | grep nvidia-operator | awk '{print $1}')
kubectl delete namespace gpu-operator
kubectl delete namespace nvidia-operator
helm uninstall gpu-operator -n nvidia-operator

kubectl get pods -n gpu-operator
Check driver installation log in pod:
kubectl exec -n gpu-operator -it $(kubectl get pod -n gpu-operator -l app=nvidia-driver-daemonset -o jsonpath="{.items[0].metadata.name}") -- cat /var/log/nvidia-installer.log

# If you face helm failed to connect to AKS cluster issue in WSL, you will need to change .kube/config 
rm -f $HOME/.kube/config
cp /mnt/c/Users/USERNAME/.kube/config $HOME/.kube/config

# If command "kubectl get nodes -o jsonpath='{.items[*].metadata.labels}' | grep nvidia.com/gpu.present" has output ","nvidia.com/gpu.present":"true" which means nvidia GPU operator is detecting gpu.
```
**References**

- https://techcommunity.microsoft.com/blog/azurehighperformancecomputingblog/deploy-ndm-v4-a100-kubernetes-cluster/3838871
- https://github.com/edwardsp/ai-on-aks
- https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html
