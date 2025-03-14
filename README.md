# Use GPU In AKS 
Use GPU in AKS cluster
This documents walks you through the steps to deploy AKS GPU cluster by using device plugin and Nvidia GPU operator separately, and to use public ip to ssh into the node to veriy the Nvidia GPU driver installed.

# Related information can be found from documents:
* [`Add Ubuntu GPU node pool`](https://learn.microsoft.com/en-us/azure/aks/gpu-cluster?tabs=add-ubuntu-gpu-node-pool)
* [`Installing Nvidia GPU Operator`](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html)
* [`GPU operator component matrix`](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html#gpu-operator-component-matrix)

# Prerequisites:
* Make sure your susbcription has enough quota for the target GPU VM SKU you are going to deploy in AKS cluster.
* Follow[`Use a public ip prefix`](https://learn.microsoft.com/en-us/azure/aks/use-node-public-ips#use-a-public-ip-prefix) to create public ip prefix so that you can use private key to easily ssh into the GPU node to verify GPU component in VM level.
 

