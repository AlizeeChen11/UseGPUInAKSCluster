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

