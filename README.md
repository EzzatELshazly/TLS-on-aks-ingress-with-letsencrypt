![lets encrypt and aks](https://github.com/user-attachments/assets/17a7eee1-5d35-4263-bfb5-db8cda31da74)

# Deploying NGINX Ingress with Cert-Manager on Azure AKS

This guide walks through the process of deploying an NGINX Ingress Controller on Azure Kubernetes Service (AKS), setting up automated SSL using Let's Encrypt via Cert-Manager, and exposing applications using a custom domain and a static public IP address.  


---

## Prerequisites

- An existing AKS cluster  
- `kubectl`, `helm`, and `az` CLI installed and configured  
- A registered custom domain  
- Access to DNS management to create A records  

---

## Step 1: Create Namespace for Ingress Resources

Run the following command:

```bash
kubectl create namespace public-ingress
```

---

## Step 2: Install NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace public-ingress \
  --set controller.config.http2=true \
  --set controller.config.http2-push="on" \
  --set controller.config.http2-push-preload="on" \
  --set controller.ingressClassByName=true \
  --set controller.ingressClassResource.controllerValue=k8s.io/ingress-nginx \
  --set controller.ingressClassResource.enabled=true \
  --set controller.ingressClassResource.name=public \
  --set controller.service.externalTrafficPolicy=Local \
  --set controller.setAsDefaultIngress=true
```

---

## Step 3: Install Cert-Manager for SSL Certificates

```bash
kubectl label namespace public-ingress cert-manager.io/disable-validation=true

helm repo add jetstack https://charts.jetstack.io
helm repo update

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml

helm install cert-manager jetstack/cert-manager \
  --namespace public-ingress \
  --version v1.11.0
```
---

## Step 4: Create a Cluster Issuer (Let’s Encrypt)

Create a file named `cluster-issuer.yaml` with the following content:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
      - http01:
          ingress:
            class: public
```

Apply the issuer:

```bash
kubectl apply -f cluster-issuer.yaml --namespace public-ingress
```
![cluster issuer](https://github.com/user-attachments/assets/c88b457d-dbb0-41fc-8636-d733d55e8f5d)

---

## Step 5: Deploy Sample Applications

Create `deployment-one.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aks-helloworld-one
  namespace: your-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: aks-helloworld-one
  template:
    metadata:
      labels:
        app: aks-helloworld-one
    spec:
      containers:
      - name: aks-helloworld-one
        image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
        ports:
        - containerPort: 80
        env:
        - name: TITLE
          value: "Welcome to my site :')"
---
apiVersion: v1
kind: Service
metadata:
  name: aks-helloworld-one
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: aks-helloworld-one
```

Apply the deployment:

```bash
kubectl apply -f dev-service-core-deployment.yaml
```
![deployment and svc](https://github.com/user-attachments/assets/641407b0-274c-45f2-9e04-ee12005f8766)

---

## Step 6: Create A Record

Get the public IP:

```bash
kubectl get svc -n public-ingress
```
![ingress load balancer ip](https://github.com/user-attachments/assets/b0428e48-bad2-4a2f-a061-aeae315b60d9)

Then navigate to your domain hosting provider (e.g., GoDaddy):  
**My Products > Choose Domain > DNS > Add New Record**  
![godady](https://github.com/user-attachments/assets/20bee140-5b29-4b71-9a8e-2afb19cd89b3)

---

## Step 7: Create Ingress Resource

Create `dev-service-core-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: site-ingress
  namespace: your-namespace
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-production
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: public
  tls:
  - hosts:
    - your-domain.com
    secretName: site-tls-secret
  rules:
  - host: your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: aks-helloworld-one
            port:
              number: 80
```

Apply the ingress:

```bash
kubectl apply -f dev-service-core-ingress.yaml
```
![ingress created](https://github.com/user-attachments/assets/0589020b-51fd-4be5-a1d1-39a5ff78554c)

---

## Check Certificate Status

```bash
kubectl get certificates -n public-ingress
```

If the certificate shows `False`, fix it by editing the ACME solver ingress:

```bash
kubectl edit ingress cm-acme-http-solver-XXXXXX --namespace your-namespace
```

In the `spec` section, add:

```yaml
ingressClassName: public
```
![edit ingress ](https://github.com/user-attachments/assets/46b82cd8-580e-4fa4-9d12-7e328622eaf5)

Wait for ~2 minutes, then re-check the certificate:

```bash
kubectl get certificate --namespace public-ingress
```
![certs](https://github.com/user-attachments/assets/acf90e3b-c38c-4159-aabe-c622777f726a)

---

## Access Your Application

You should now be able to access your site securely:

- `https://your-domain.com`  
![access the site](https://github.com/user-attachments/assets/78d8323f-8b94-4855-9fe6-4c621efbab70)

---

Now your AKS applications are live with HTTPS using Let's Encrypt and accessible via your custom domain.
