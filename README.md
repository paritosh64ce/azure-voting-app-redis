---
page_type: sample
languages:
  - python
products:
  - azure
  - azure-redis-cache
description: "This sample creates a multi-container application in an Azure Kubernetes Service (AKS) cluster."
---

# Azure Voting App

This sample creates a multi-container application in an Azure Kubernetes Service (AKS) cluster. The application interface has been built using Python / Flask. The data component is using Redis.

To walk through a quick deployment of this application, see the AKS [quick start](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough?WT.mc_id=none-github-nepeters).

To walk through a complete experience where this code is packaged into container images, uploaded to Azure Container Registry, and then run in and AKS cluster, see the [AKS tutorials](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-app?WT.mc_id=none-github-nepeters).

## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

---

## Commands

#### [Reference: Azure K8s Service](https://learn.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-app)

- Create resource group and Container registry

  ``` cmd
  az group create --name myResourceGroup --location centralindia  
  az acr create --resource-group myResourceGroup --name <acrName> --sku Basic
  ```

- Get login server name

  ``` cmd
  az acr list --resource-group myResourceGroup --query "[].{acrLoginServer:loginServer}" --output table
  ```

- Tag docker image for your Azure Container Registry, push it and get repository list to validate
  
  ``` cmd
  docker tag mcr.microsoft.com/azuredocs/azure-vote-front:v1 <acrLoginServer>/azure-vote-front:v1  
  docker images  
  docker push <acrLoginServer>/azure-vote-front:v1  
  az acr repository list --name <acrName> --output table
  ```

- Create K8s cluster and attach it with your ACR

  ``` cmd
  az aks create \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --node-count 2 \
    --generate-ssh-keys \
    --attach-acr <acrName>
  ```

- Get credentials for your AKS
  
  ```
  az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
  > Merged "myAKSCluster" as current context in C:\Users\<User>\.kube\config
  
  kubectl get nodes
  > NAME                                STATUS   ROLES   AGE   VERSION  
  > aks-nodepool1-33106043-vmss000000   Ready    agent   23m   v1.24.6  
  > aks-nodepool1-33106043-vmss000001   Ready    agent   23m   v1.24.6  
  ```

- Deploy the application
  ``` cmd
  kubectl apply -f azure-vote-all-in-one-redis.yaml
  ```

- Get IP address
  ``` cmd
  kubectl get service azure-vote-front --watch
  > NAME               TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE  
  > azure-vote-front   LoadBalancer   10.0.253.123   20.204.188.46   80:31563/TCP   45s  
  ```

- Manually scale pods
  ```
  kubectl get pods
  > NAME                                READY   STATUS    RESTARTS        AGE  
  > azure-vote-back-5fb9656dff-zk4sx    1/1     Running   0               3m42s  
  > azure-vote-front-7fbd799d66-fmkn4   1/1     Running   1 (3m13s ago)   3m42s

  kubectl scale --replicas=5 deployment/azure-vote-front
  > Warning: spec.template.spec.nodeSelector[beta.kubernetes.io/os]: deprecated since v1.14; use "kubernetes.io/os" instead deployment.apps/azure-vote-front scaled
  
  kubectl get pods
  > NAME                                READY   STATUS              RESTARTS       AGE  
  > azure-vote-back-5fb9656dff-zk4sx    1/1     Running             0              5m34s  
  > azure-vote-front-7fbd799d66-2m2ql   0/1     ContainerCreating   0              7s  
  > azure-vote-front-7fbd799d66-96lml   0/1     ContainerCreating   0              7s  
  > azure-vote-front-7fbd799d66-f4rdp   1/1     Running             0              7s  
  > azure-vote-front-7fbd799d66-fmkn4   1/1     Running             1 (5m5s ago)   5m34s  
  > azure-vote-front-7fbd799d66-pk9qn   1/1     Running             0              7s  
  ```

- Autoscale

  ``` cmd
  kubectl autoscale deployment azure-vote-front --cpu-percent=50 --min=3 --max=10
  > horizontalpodautoscaler.autoscaling/azure-vote-front autoscaled  

  kubectl apply -f azure-vote-hpa.yaml
  > horizontalpodautoscaler.autoscaling/azure-vote-back-hpa created  
  > horizontalpodautoscaler.autoscaling/azure-vote-front-hpa created

  kubectl get hpa
  > NAME                   REFERENCE                     TARGETS         MINPODS   MAXPODS   REPLICAS   AGE  
  > azure-vote-back-hpa    Deployment/azure-vote-back    <unknown>/50%   3         10        3          2m1s  
  > azure-vote-front       Deployment/azure-vote-front   0%/50%          3         10        3          4m59s  
  > azure-vote-front-hpa   Deployment/azure-vote-front   0%/50%          3         10        3          2m1s  
  ```

- Manually scale AKS nodes

  ``` cmd
  az aks scale --resource-group myResourceGroup --name myAKSCluster --node-count 3
  > (CreateVMSSAgentPoolFailed) Code="OperationNotAllowed" Message="Operation could not be completed as it results in exceeding approved Total Regional Cores quota. Additional details - Deployment Model: Resource Manager, Location: CentralIndia, Current Limit: 4, Current Usage: 4, Additional Required: 2, (Minimum) New Limit Required: 6. Submit a request for Quota increase at https://aka.ms/ProdportalCRP/#blade/Microsoft_Azure_Capacity/UsageAndQuota.ReactView/Parameters/%7B%22subscriptionId%22:%22<SubscriptionId>%22,%22command%22:%22openQuotaApprovalBlade%22,%22quotas%22:[%7B%22location%22:%22CentralIndia%22,%22providerId%22:%22Microsoft.Compute%22,%22resourceName%22:%22cores%22,%22quotaRequest%22:%7B%22properties%22:%7B%22limit%22:6,%22unit%22:%22Count%22,%22name%22:%7B%22value%22:%22cores%22%7D%7D%7D%7D]%7D by specifying parameters listed in the ‘Details’ section for deployment to succeed. Please read more about quota limits at https://docs.microsoft.com/en-us/azure/azure-supportability/regional-quota-requests"  
  > Code: CreateVMSSAgentPoolFailed
  > Message: Code="OperationNotAllowed" Message="Operation could not be completed as it results in exceeding approved Total Regional Cores quota. Additional details - Deployment Model: Resource Manager, Location: CentralIndia, Current Limit: 4, Current Usage: 4, Additional Required: 2, (Minimum) New Limit Required: 6. Submit a request for Quota increase at https://aka.ms/ProdportalCRP/#blade/Microsoft_Azure_Capacity/UsageAndQuota.ReactView/Parameters/%7B%22subscriptionId%22:%22<SubscriptionId>%22,%22command%22:%22openQuotaApprovalBlade%22,%22quotas%22:[%7B%22location%22:%22CentralIndia%22,%22providerId%22:%22Microsoft.Compute%22,%22resourceName%22:%22cores%22,%22quotaRequest%22:%7B%22properties%22:%7B%22limit%22:6,%22unit%22:%22Count%22,%22name%22:%7B%22value%22:%22cores%22%7D%7D%7D%7D]%7D by specifying parameters listed in the ‘Details’ section for deployment to succeed. Please read more about quota limits at https://docs.microsoft.com/en-us/azure/azure-supportability/regional-quota-requests"
  ```

- Update application

  - Update config file -> azure-vote/azure-vote/config_file.cfg
  ``` cmd
  docker-compose build
  docker tag mcr.microsoft.com/azuredocs/azure-vote-front:v1 <acrLoginServer>/azure-vote-front:v2
  docker push <acrLoginServer>/azure-vote-front:v2
  ```

- Validate how many replicas are there

  ``` cmd
  kubectl get pods  
  kubectl scale --replicas=3 deployment/azure-vote-front  
  kubectl get pods  
  ```

- Update image and validate the progress of deployment

  ``` cmd
  kubectl set image deployment azure-vote-front azure-vote-front=<acrLoginServer>/azure-vote-front:v2  
  kubectl get pods
  ```

- Check the IP address and validate the application

  ``` cmd
  kubectl get service azure-vote-front
  ```

- Upgrade Kubernetes in Azure Kubernetes Service (AKS)

  ``` cmd
  az aks get-upgrades --resource-group rg-aks --name myAKSCluster
  ```

- Don't forget to delete all the resources and resource-group to avoid the cost!!!
  ``` cmd
  az group delete --name rg-aks --yes --no-wait
  ```
