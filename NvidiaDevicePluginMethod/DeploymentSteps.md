
# Create aks cluster specifying public ip and ssh key enabled:
az aks create -g ResourceGroupName -n AKSClusterName --nodepool-name AgentPoolName --node-count 1 --enable-node-public-ip --node-public-ip-prefix ResourceIdOfPublicIpPrefix --ssh-key-value /path/to/publicKey/*.pub --location southcentralus

# Get AKS ckuster credential:
az aks get-credentials --resource-group ResourceGroupName --name AKSClusterName

# Add GPU Node pool:
az aks nodepool add --resource-group ResourceGroupName --cluster-name AKSClusterName --name GPUNodePoolName --node-count 1 --node-vm-size Standard_ND96asr_v4  --enable-node-public-ip --node-public-ip-prefix ResourceIdOfPublicIpPrefix

# Create GPU resource namespace:
kubectl create namespace gpu-resources

# Install Nvidia device plugin version=0.14.5-3:
kubectl apply -f nvidia-device-plugin-ds-145-3.yaml

# Get GPU node name:
kubectl get nodes

# Verify we have GPU device schedulable in "Allocatable" part and "Allocated resources":
kubectl describe node GPUNodeName

# Find the AKS VMSS resoruce group name:
az aks show -g ResourceGroupName -n AKSClusterName --query nodeResourceGroup

# Get the GPU nodepool VMSS name: 
az vmss list -g <nodeGroupName> --query [].name

# list AKS cluster GPU nodepool vmss ip:
az vmss list-instance-public-ips --resource-group MC_dandchen-cycle82rg_elsaakscluster2_southcentralus --name GPUVMSSname

# Use private key to ssh into the GPU node verify the Nvidia driver version:
ssh -i *.pem username@PublicIPOfGPUNode
