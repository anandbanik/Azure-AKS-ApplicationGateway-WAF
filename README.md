# Azure-AKS-ApplicationGateway-WAF

This documentation explains how you can configure your kubernetes cluster behind Application Gateway and Web Application Firewall on Azure Portal.

You can also checkout the [YouTube video](https://www.youtube.com/watch?v=GkznBbNSq7U&t=316s) for visual explanation.

You would have had an understanding of configuring a LoadBalancer, WAF, AppGateway and Provisioning Kubernetes cluster
I will use the Azure Vote application which can be found on the Azure documentation website for AKS as the example which is pretty straight forward. https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-app

Steps: 

Enabling AKS preview for your Azure subscription 

    az provider register -n Microsoft.ContainerService 

Create a resource group 

    az group create --name myResourceGroup --location westeurope 

Create AKS cluster 

    az aks create --resource-group myResourceGroup --name myAKSCluster --node-count 1 --generate-ssh-keys 

Connect to the cluster 

    az aks install-cli 
    az aks get-credentials --resource-group myResourceGroup --name myAKSCluster 
    kubectl get nodes 

## External L4 LoadBalancer
Download the Azure Vote YAML file (azure-vote-all-in-one-redis.yaml) from

    https://github.com/Azure-Samples/azure-voting-app-redis 

Use the kubectl create command to run the application. 
```
kubectl create -f azure-vote-all-in-one-redis.yaml 
```

You can now browse to the external IP address to see the Azure Vote App 

## Internal L4 LoadBalancer:

To do this, I will be adding the following annotations to the service based on cloud provider to the azure-vote-all-in-one-redis.yaml file 

[...] 
metadata: 
    name: my-service 
    annotations: 
        service.beta.kubernetes.io/azure-load-balancer-internal: "true" 
[...] 
  

The "azure-vote-front" service config in the azure-vote-all-in-one-redis.yaml file will then look like this: 
  
```
--- 
apiVersion: v1 
kind: Service 
metadata: 
  name: azure-vote-front 
  annotations: 
      service.beta.kubernetes.io/azure-load-balancer-internal: "true" 
spec: 
  type: LoadBalancer 
  ports: 
  - port: 80 
  selector: 
    app: azure-vote-front 
``` 

If you run the get service command; it will look like this

![alt text](/images/GetSVC2.PNG "Get Services")

The resources in the resource group will now look like this
![alt text](/images/CreatedAPG.PNG "Azure resources")


# Exposing the App through the Application Gateway & Web Application Firewall

There are 2 methods I will describe here to achieve the picture below:
![alt text](/images/APG.PNG "Creating APG")

# Different VNET

## Create the AppGateway/WAF

Create the Gateway new or different Resource group and VNET

![alt text](/images/CreatingAGW1-Method1.PNG "Creating APG")

![alt text](/images/CreatingAGW2-Method1.PNG "Creating APG")

Add the IP Address of the LoadBalancer as the backend IP of my pre-configured AppGatway/WAF. 


Then peer the VNET of the cluster resource group and the AppGateway resource group.

![alt text](/images/Peering1.PNG "Creating APG")

## Peer from K8s Cluster VNET to the AppGateway VNET
![alt text](/images/PeeringtoK8s.PNG "Creating APG")

## Peer from K8s Cluster VNET to the AppGateway VNET
![alt text](/images/PeeringtoAPG.PNG "Creating APG")

You can now browse to the external IP address of the Application Gateway to see the Azure Vote App 

![alt text](/images/APG2.PNG "Files, folders and naming conventions")

# Same VNET	

## Create the AppGateway/WAF

## Create the Subnet in the VNET of the K8s Cluster

![alt text](/images/GatewaySubnet.PNG "Creating Subnet")

![alt text](/images/GatewaySubnet2.PNG "Creating Subnet")

## Create the AppGateway/WAF in the same resource group of the cluster and use the same VNET

![alt text](/images/CreatingAGW1.PNG "Creating APG")

![alt text](/images/CreatingAGW2.PNG "Creating APG")

You can now browse to the external IP address of the Application Gateway to see the Azure Vote App 

![alt text](/images/APG1.PNG "Files, folders and naming conventions")

# Ingress

Another use case can be to have the loadbalancer to route traffic to more apps in different services endpoint running in the same cluster. In this case, we'll use the Ingress service to manage the traffic routing for the cluster  ingress traffic and proxying it to the right endpoints. 

Ingress can provide load balancing, SSL termination and name-based virtual hosting. 

Please visit the kubernetes documentation to learn more on setting up an Ingress Controller 

DeployIngress.yaml for a single Service Ingress 
```
--- 
apiVersion: extensions/v1beta1  
kind: Ingress  
metadata:  
  name: nginx-ingress  
spec:  
  backend:   
  serviceName: nginx  
  ServicePort: 80  

For multiple endpoints, the file will look like this 

DeployIngress.yaml 
```
```
--- 
apiVersion: extensions/v1beta1 
kind: Ingress 
metadata: 
  name: test 
  annotations: 
    ingress.kubernetes.io/rewrite-target: / 
spec: 
  rules: 
  - host: foo.bar.com 
    http: 
      paths: 
      - path: /foo 
        backend: 
          serviceName: s1 
          servicePort: 80 
      - path: /bar 
        backend: 
          serviceName: s2 
          servicePort: 80 
  
```
