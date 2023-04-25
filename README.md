# Oefenpraktijk Azure Kuberenetes Service Setup

# Why Kubernetes?

The main incentive to use Kubernetes for this project is learning. There are more simple alternatives, however, in my opinion, it is more beneficial to understand how to set up and configure the Kubernetes cluster. Additionally, our client Susan indicated that the budgets for the beginning stage of the project are rather low, and managed Kubernetes is a more affordable option than solutions like Azure App Service.

# Why Microsoft Azure?

We inherited the Kubernetes cluster in Azure from the previous team and we decided that it makes sense to keep it since we won't have to create a new cluster ourselves. Fontys has an Azure environment and we operate in it under our resource group. That means that Fontys also covers the bills.

# Foundations

At the moment we run the minimal operatable configuration of the Kubernetes cluster with just 2 B2ms nodes so that we can cut costs as much as possible during development. For the future, we have two node pools configured:

- Agent pool (agentpool) – infrastructure nodes (only k8s-related containers)
- User pool (userpool) – for workloads

When the cluster is in the production stage, I would recommend keeping the workloads (e.g. APIs) segregated from infrastructure nodes. This will assure that nothing throttles the control plane (no noisy neighbours for infra nodes).

# Infrastructure as Code

I created a separate repository ([https://github.com/De-Oefenpraktijk/Infra-As-Code](https://github.com/De-Oefenpraktijk/Infra-As-Code)) that contains all the **yaml** files for the deployment of the application and a set of instructions for cluster deployment.

I used Kustomize ([https://kustomize.io/](https://kustomize.io/)) manifest so that you can apply the whole application with just one command ( **kubectl apply -k .** ) in the root of the folder. Kustomize will apply all the yamls in the correct order.

# Secret Management

If you check any of the deployment yamls you will see that the environmental variables are not passed directly to the container but are referenced from the **secret** object.

      env:
        - name: EventBusSettings__HostAddress
          valueFrom:
            secretKeyRef:
              name: env-secret
              key: eventbus

The secret itself is only provided as a template and does not contain any sensitive data. In the case of real-world deployment, you need to fill in the actual values. It is safe to use it this way since Kubernetes secrets are also encrypted on the VM level  (_How to Create and Use Kubernetes Secrets_, n.d.).

# Networking

## Research

Networking in Azure turned out to be full of complex concepts for me. I used Design Pattern Research and Literature Study from DOT research methods to find the list of options and analyze them properly.

Essentially, all the traffic that comes to the Kubernetes cluster needs to be navigated to a certain resource. That can be done with the ingress resource. [Ingress](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.27/#ingress-v1-networking-k8s-io) exposes HTTP and HTTPS routes from outside the cluster to [services](https://kubernetes.io/docs/concepts/services-networking/service/) within the cluster. (Ingress, n.d.).An Ingress Controller is a component in a Kubernetes cluster that configures an HTTP load balancer according to Ingress resources created by the cluster user. (How NGINX Ingress Controller Works | NGINX Ingress Controller, n.d.)

To summarize, these are the possible options for setting up networking for the AKS cluster:

1. [Ingress resources using NGINX](https://learn.microsoft.com/en-us/azure/aks/ingress-basic)

NGNIX is built around Kubernetes Ingress Resource and is one possible implementation of the ingress controller for the Kubernetes cluster. In this case, the traffic that comes to the Azure LoadBalancer in front of the cluster is redirected to the ingress controller service of type LoadBalancer and further on to the services according to ingress rules. That is a manual but highly configurable way to set up ingress.

2. [HTTP application routing](https://learn.microsoft.com/en-us/azure/aks/http-application-routing)

HTTP application add-on simplifies the deployment of the ingress controller. It also creates a publicly accessible DNS name for the IP address. However, this addon doesn't work with Kubernetes 1.22+ and thus is not suitable for our case. It is also not recommended for production workloads.

3. [Web Application Routing](https://learn.microsoft.com/en-us/azure/aks/web-app-routing)

Web application routing addon is similar to HTTP application routing addon. However, it is created for production workloads. Under the neath it relies on the ngnix ingress controller and it's effectively a wrapper around it. However, it's currently in a preview state and is not yet recommended in production workloads.

4. [Application Gateway Ingress Controller](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-overview)

Application Gateway Ingress Controller makes it possible to leverage Azure Application Gateway for Azure Kubernetes Service. The idea is that it deploys both the ingress controller inside of the cluster and an Azure Application Gateway upfront the cluster. Then it translates Kubernetes ingress resources into Azure Application Gateway rules. The main benefit is that it can react in response to an increase or decrease in traffic load and scale accordingly, without consuming any resources from your AKS cluster.(learn.microsoft.com, n.d.)

## Prototype

I deployed the ngnix ingress controller to our production cluster and spined up another small cluster with Application Gateway Ingress Controller (AGIC). Both solutions worked after some configurations, however, one major difference is that the ngnix ingress controller is much more customizable and supports many more annotations( For example, nginx.ingress.kubernetes.io/rewrite-target)

Another difference is that from what I found, AGIC doesn't support regular expressions for path configuration, which is needed in our case.

## Conclusion

Overall, I decided to use the NGNIX ingress controller because of its high flexibility and stability. Its main feature is the ability to create a rewrite annotation to append an additional path to the domain.

# Architecture

![image](https://www.stacksimplify.com/course-images/azure-aks-ingress-basic.png)

[_https://stacksimplify.com/azure-aks/azure-kubernetes-service-ingress-basics/_](https://stacksimplify.com/azure-aks/azure-kubernetes-service-ingress-basics/)

As you can see in the diagram, there is an AKS Load Balancer upfront in the cluster that can be accessed via the public IP. Then it forwards the traffic according to the external port of the ingress controller’s service of type Load Balancer. The Ingress controller then approaches the ingress resource and in accordance with the defined ingress rule the traffic is transferred to the service of the application and then to the pod. 

We use a single ingress rule resource that defines the rules for all services. We also use public domain from Azure, namely oefenpraktijkapi.westeurope.cloudapp.azure.com . 


Microservice can be accessed in a following way:

oefenpraktijkapi.westeurope.cloudapp.azure.com/\<microservice\>/…

Example:

[https://oefenpraktijkapi.westeurope.cloudapp.azure.com/profile/api/v1/Education/GetEducations](https://oefenpraktijkapi.westeurope.cloudapp.azure.com/profile/api/v1/Education/GetEducations)

## TLS termination

We use TLS termination for our endpoints, meaning that ingress protects the path by SSL, however, communication between ingress and service happens via HTTP because the traffic is inside of the cluster. 

# References

How to Create and Use Kubernetes Secrets. (n.d.). Sysdig. Retrieved April 25, 2023, from [https://sysdig.com/learn-cloud-native/kubernetes-101/how-to-create-and-use-kubernetes-secrets/](https://sysdig.com/learn-cloud-native/kubernetes-101/how-to-create-and-use-kubernetes-secrets/)

Ingress. (n.d.). Kubernetes. [https://kubernetes.io/docs/concepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/)

How NGINX Ingress Controller Works | NGINX Ingress Controller. (n.d.). Docs.nginx.com. [https://docs.nginx.com/nginx-ingress-controller/intro/how-nginx-ingress-controller-works/](https://docs.nginx.com/nginx-ingress-controller/intro/how-nginx-ingress-controller-works/)

greg-lindsay. (n.d.). What is Azure Application Gateway Ingress Controller? Learn.microsoft.com. Retrieved April 25, 2023, from [https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-overview](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-overview)
