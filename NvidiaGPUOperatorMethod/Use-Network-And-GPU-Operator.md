# Use Network and GPU Operator for GPUDirect RDMA in AKS cluster
This document inlcudes steps to apply Nvidia Network Operator for MOFED driver and Nvidia GPU Operator for GPU driver in AKS GPU nodepool.

## Installation
### Prerequisites
- Azure subscription with GPU quota(example is using NDA100v4)
- A Client with Azure cli installed
- WSL for Windows

### Deploy Azure resources
Install kubectl:
az extension add --name aks-preview
az aks install-cli

Deploy AKS cluster and add GPU node pool:
```
az aks create -g ResourceGroupName -n AKSClusterName --ssh-key-value SSHKEYPATH --location REGION
az aks get-credentials --resource-group ResourceGroupName --name AKSClusterName
az aks nodepool add --resource-group ResourceGroupName --cluster-name AKSClusterName --name GPUNodePoolName --node-count 1 --node-vm-size Standard_ND96asr_v4  --tags SkipGPUDriverInstall=True --enable-node-public-ip --node-public-ip-prefix ResourceIdOfPublicIpPrefix
```
Note: Enable gpu node public ip is for debugging purpose. If there is pod failure, we can ssh into the node to check pod log.

Install Helm in WSL:
Make sure the kube config file is in the correct path:
```
cp /mnt/c/Users/USERNAME/.kube/config ${HOME}/.kube/
wget https://get.helm.sh/helm-v3.17.2-linux-amd64.tar.gz
tar -zxvf helm-v3.0.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia 
helm repo update
helm install --wait --generate-name -n gpu-operator --create-namespace nvidia/gpu-operator
```

### Additional commands for troublehsooting purpose:
```
Remove gpu-operatore steps(run in WSL):
helm delete -n nvidia-operator $(helm list -n nvidia-operator | grep nvidia-operator | awk '{print $1}')
kubectl delete namespace gpu-operator
kubectl delete namespace nvidia-operator
helm uninstall gpu-operator -n nvidia-operator

kubectl get pods -n gpu-operator
Check driver installation log in pod:
kubectl exec -n gpu-operator -it $(kubectl get pod -n gpu-operator -l app=nvidia-driver-daemonset -o jsonpath="{.items[0].metadata.name}") -- cat /var/log/nvidia-installer.log

Note: If you face helm failed to connect to AKS cluster issue in WSL, you will need to change .kube/config 
az aks get-credentials --resource-group dandchen-jerg --name elsaakscluster1 --overwrite-existing
rm -f /home/azureuser/.kube/config
cp /mnt/c/Users/USERNAME/.kube/config $HOME/.kube/config
```

**References**

- https://techcommunity.microsoft.com/blog/azurehighperformancecomputingblog/deploy-ndm-v4-a100-kubernetes-cluster/3838871
- https://github.com/edwardsp/ai-on-aks
- 


