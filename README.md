Deploying NGINX Ingress with Cert-Manager on Azure AKS

This guide walks through the process of deploying an NGINX Ingress Controller on Azure Kubernetes Service (AKS), setting up automated SSL using Let's Encrypt via Cert-Manager, and exposing applications using a custom domain and a static public IP address.
# **Prerequisites**
\- An existing AKS cluster
\- kubectl, helm, and az CLI installed and configured
\- A registered custom domain
\- Access to DNS management to create A records
# **Step 1: Create Namespace for Ingress Resources**
Run the following command:

***kubectl create namespace public-ingress***
# **Step 2: Install NGINX Ingress Controller**

***helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx \
`  `--repo https://kubernetes.github.io/ingress-nginx \
`  `--namespace public-ingress \
`  `--set controller.config.http2=true \
`  `--set controller.config.http2-push="on" \
`  `--set controller.config.http2-push-preload="on" \
`  `--set controller.ingressClassByName=true \
`  `--set controller.ingressClassResource.controllerValue=k8s.io/ingress-nginx \
`  `--set controller.ingressClassResource.enabled=true \
`  `--set controller.ingressClassResource.name=public \
`  `--set controller.service.externalTrafficPolicy=Local \
`  `--set controller.setAsDefaultIngress=true***

# **Step 3: Install Cert-Manager for SSL Certificates**

***kubectl label namespace public-ingress cert-manager.io/disable-validation=true

helm repo add jetstack https://charts.jetstack.io
helm repo update

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml

helm install cert-manager jetstack/cert-manager \
`  `--namespace public-ingress \
`  `--version v1.11.0***

# **Step 4: Create a Cluster Issuer (Let’s Encrypt)**
Create a file named cluster-issuer.yaml with the following content:


***apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
`  `name: letsencrypt-production
spec:
`  `acme:
`    `server: https://acme-v02.api.letsencrypt.org/directory
`    `email: # your-email@example.com
`    `privateKeySecretRef:
`      `name: letsencrypt-production
`    `solvers:
`      `- http01:
`          `ingress:
`            `class: public***


Apply the issuer:

***kubectl apply -f cluster-issuer.yaml --namespace public-ingress***
# **Step 5: Deploy Sample Applications**
Create deployment-one.yaml:

***apiVersion: apps/v1***

***kind: Deployment***

***metadata:***

`  `***name: aks-helloworld-one
` `namespace: #your-namespace***

***spec:***

`  `***replicas: 1***

`  `***selector:***

`    `***matchLabels:***

`      `***app: aks-helloworld-one***

`  `***template:***

`    `***metadata:***

`      `***labels:***

`        `***app: aks-helloworld-one***

`    `***spec:***

`      `***containers:***

`      `***- name: aks-helloworld-one***

`        `***image: mcr.microsoft.com/azuredocs/aks-helloworld:v1***

`        `***ports:***

`        `***- containerPort: 80***

`        `***env:***

`        `***- name: TITLE***

`          `***value: "Welcome to my site :')"***

***---***

***apiVersion: v1***

***kind: Service***

***metadata:***

`  `***name: aks-helloworld-one***

***spec:***

`  `***type: ClusterIP***

`  `***ports:***

`  `***- port: 80***

`  `***selector:***

`    `***app: aks-helloworld-one***

![](Aspose.Words.9b6a2fb2-a888-4efb-8055-e2569fbba5c3.001.png)
***Apply the deployment:
Kubectl apply -f dev-service-core-deployment.yaml*** 

**Step 6: Create A record** 
A record of custom domain will be the Public IP of Ingress Controller.
kubectl get svc -n public-ingress
![A screen shot of a computer

AI-generated content may be incorrect.](Aspose.Words.9b6a2fb2-a888-4efb-8055-e2569fbba5c3.002.png)
Then Navigate to your domain hosting; I use GoDaddy  
=================================================================================================================================
![A screenshot of a computer

AI-generated content may be incorrect.](Aspose.Words.9b6a2fb2-a888-4efb-8055-e2569fbba5c3.003.png)My products> choose your domain> domain > DNS > add new record
# **Step 7: Create ingress resource** 
***apiVersion: networking.k8s.io/v1***

***kind: Ingress***

***metadata:***

`  `***name: site-ingress
`  `namespace: #your-namespace***

`  `***annotations:***

`    `***cert-manager.io/cluster-issuer: letsencrypt-production***

`    `***nginx.ingress.kubernetes.io/rewrite-target: /***

***spec:***

`  `***ingressClassName: public***

`  `***tls:***

`  `***- hosts:***

`    `***- # add your domain name here***

`    `***secretName: site-tls-secret***

`  `***rules:***

`  `***- host: # add your domain name here***

`    `***http:***

`      `***paths:***

`      `***- path: /***

`        `***pathType: Prefix***

`        `***backend:***

`          `***service:***

`            `***name: aks-helloworld-one***

`            `***port:***

`              `***number: 80***

Apply the ingress:
kubectl apply -f dev-service-core-ingress.yaml
![](Aspose.Words.9b6a2fb2-a888-4efb-8055-e2569fbba5c3.004.png)

Now check the certificates 

Kubectl get certificates -n public-ingress

If the certificate is in a false state, to solve this:
After creating the ingress , another ingress is created acem-solver ingress. 
so you need to edit this ingress and add ingressClassName in spec section.

Kubectl edit ingress cm-acme-http-solver-379sq –namespace #yournamespace

![A screenshot of a computer

AI-generated content may be incorrect.](Aspose.Words.9b6a2fb2-a888-4efb-8055-e2569fbba5c3.005.png)

Wait for two minutes approximately you will find that the certificate is changed to true.
kubectl get certificate --namespace public-ingress

![A black screen with white text

AI-generated content may be incorrect.](Aspose.Words.9b6a2fb2-a888-4efb-8055-e2569fbba5c3.006.png)

Try now access your site with you domain.

![A screenshot of a computer

AI-generated content may be incorrect.](Aspose.Words.9b6a2fb2-a888-4efb-8055-e2569fbba5c3.007.png)
