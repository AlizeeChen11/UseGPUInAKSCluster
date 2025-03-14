# Prerequisites
Please make sure you have performed the prerequisites part.

# Detailed steps:
## Create aks cluster specifying public ip and ssh key enabled:
```
az aks create -g ResourceGroupName -n AKSClusterName --nodepool-name AgentPoolName --node-count 1 --enable-node-public-ip --node-public-ip-prefix ResourceIdOfPublicIpPrefix --ssh-key-value /path/to/publicKey/*.pub --location southcentralus
```

## Get AKS ckuster credential:
```
az aks get-credentials --resource-group ResourceGroupName --name AKSClusterName
```

## Add GPU Node pool:
```
az aks nodepool add --resource-group ResourceGroupName --cluster-name AKSClusterName --name GPUNodePoolName --node-count 1 --node-vm-size Standard_ND96asr_v4 --node-taints sku=gpu:NoSchedule --enable-node-public-ip --node-public-ip-prefix ResourceIdOfPublicIpPrefix
```

## Skip driver installation:
```
az aks nodepool update --resource-group ResourceGroupName --cluster-name AKSClusterName --name gpunp4 --tags SkipGPUDriverInstall=True
```

## Stop GPUNodePoolName nodepool, then start to let the SkipGPUDriverInstall=True taking effect n newly created node:
```
az aks nodepool stop --resource-group ResourceGroupName --cluster-name AKSClusterName --nodepool-name GPUNodePoolName
az aks nodepool start --resource-group ResourceGroupName --cluster-name AKSClusterName --nodepool-name GPUNodePoolName
```

## Installing the NVIDIA GPU Operator:
```
curl https://get.helm.sh/helm-v3.15.1-windows-amd64.zip --output helm-v3.15.1-windows-amd64.zip
mkdir helm-v3.15.1-windows-amd64
tar -xf helm-v3.15.1-windows-amd64.zip 
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia 
helm repo update
helm install --wait --generate-name -n gpu-operator --create-namespace nvidia/gpu-operator
```

## Alternatively, you can specifying a specific driver verison to install by using helm:
```
Before using helm installing gpu driver, perform below on the node to install nvidia-container-runtime:
	a. modify file sudo vi /etc/containerd/config.toml 
	b. install nvidia-container-runtime:
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

helm upgrade --install gpu-operator nvidia/gpu-operator  -n gpu-operator --create-namespace --set driver.version=525-5.15.0-1057-azure --set operator.runtimeClass=nvidia-container-runtime  --set driver.repository=nvcr.io/nvidia  --set daemonsets.tolerations[0].effect=NoSchedule --set daemonsets.tolerations[0].operator=Equal --set daemonsets.tolerations[0].key=sku --set daemonsets.tolerations[0].value=gpu  --set node-feature-discovery.worker.tolerations[0].key=node-role.kubernetes.io/master --set node-feature-discovery.worker.tolerations[0].operator=Equal   --set node-feature-discovery.worker.tolerations[0].effect=NoSchedule --set node-feature-discovery.worker.tolerations[1].key=node-role.kubernetes.io/control-plane  --set node-feature-discovery.worker.tolerations[1].operator=Equal  --set node-feature-discovery.worker.tolerations[1].effect=NoSchedule  --set node-feature-discovery.worker.tolerations[2].key=sku  --set node-feature-discovery.worker.tolerations[2].operator=Equal  --set node-feature-discovery.worker.tolerations[2].effect=NoSchedule  --set node-feature-discovery.worker.tolerations[2].value=gpu 
```

# Verify the GPU driver version:
## Get GPU node name:
```
kubectl get nodes
```

## Verify we have GPU device schedulable in "Allocatable" part and "Allocated resources":
```
kubectl describe node GPUNodeName
```

## Find the AKS VMSS resoruce group name:
```
az aks show -g ResourceGroupName -n AKSClusterName --query nodeResourceGroup
```

## Get the GPU nodepool VMSS name:
```
az vmss list -g <nodeGroupName> --query [].name
```

## list AKS cluster GPU nodepool vmss ip:
```
az vmss list-instance-public-ips --resource-group MC_dandchen-cycle82rg_elsaakscluster2_southcentralus --name GPUVMSSname
```

## Use private key to ssh into the GPU node verify the Nvidia driver version:
```
ssh -i *.pem username@PublicIPOfGPUNode
nvidia-smi
```

## Additional info:
Remove gpu-operatore steps(run in WSL):
helm delete -n gpu-operator $(helm list -n gpu-operator | grep gpu-operator | awk '{print $1}')
kubectl get pods -n gpu-operator
kubectl delete namespace gpu-operator

Check log:
kubectl get nodes
kubectl describe node xxx

kubectl get pod -n gpu-operator
kubectl describe pod <pod-name> -n gpu-operator

![image](https://github.com/user-attachments/assets/f22f66d6-e268-4041-9080-d503c778bb02)
