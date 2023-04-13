# Deploying and Packaging Applications into Kubernetes

## Deploy Jfrog Artifactory into Kubernetes

The best approach to easily get Artifactory into kubernetes is to use helm.
Search for an official helm chart for Artifactory on Artifact Hub

Add the jfrog remote repository on your laptop/computer

```bash
helm repo add jfrog https://charts.jfrog.io
```

Create namespace for the deployment

```bash
kubectl create ns tools
```

Update the helm repo index on your laptop/computer

```bash
helm repo update
```

![pods](/images/1.png)

Install artifactory chart

```bash
helm upgrade --install artifactory jfrog/artifactory --version 107.38.10 -n tools
```

![pods](/images/2.png)

Getting the Artifactory URL

The artifactory helm chart comes bundled with the Artifactory software, a PostgreSQL database and an Nginx proxy which it uses to configure routes to the different capabilities of Artifactory. Getting the pods after some time, you should see something like the below.

```bash
k get po -n tools 
```

![pods](/images/3.png)

Get the service of each deployed application and login with the load balancer exernal IP

```bash
k get svc -n tools
```

![pods](/images/4.png)

Login with the default authentication details username: *admin* password: *password*

![pods](/images/5.png)

### Deploying Ingress controller 

An ingress is an API object that manages external access to the services in a kubernetes cluster. It is capable to provide load balancing, SSL termination and name-based virtual hosting.

**Deploy Nginx Ingress Controller**

You view the offical documentation [here](https://kubernetes.github.io/ingress-nginx/deploy/)

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
  ```

![pods](/images/6.png)

  Verify ingress-nginx namespace is created

  ```bash
  kubectl get pods --namespace=ingress-nginx
  ```

![pods](/images/7.png)

Verify ingress Load balance 

![pods](/images/8.png)

The ingress controller Load balancer is ready but nothing as been deployed to it yet.

**Deploy Artifactory Ingress**

Now, it is time to configure the ingress so that we can route traffic to the Artifactory internal service, through the ingress controller’s load balancer.

Create an ingress manifest file for Artifactory as shown below

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: artifactory
spec:
  ingressClassName: nginx
  rules:
  - host: "tooling.artifactory.oayanda.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: artifactory
            port:
              number: 8082
```

```bash
# Apply 
k apply -f artifactory.yaml -n tools

# Verify
k get ingress -n tools
```

Verify
![pods](/images/9.png)

- CLASS – The nginx controller class name nginx
- HOSTS – The hostname to be used in the browser tooling.artifactory.sandbox.svc.darey.io
- ADDRESS – The loadbalancer address that was created by the ingress controller

**Configure DNS**

Create a DNS hosted zone in AWS route53 for your domain, my case *oayanda.com* and create a A record *tooling.artifactory* that points to the ingress load balancer

![pods](/images/10.png)

Let's verify this in the browser
![pods](/images/11.png)

Get the default login

```bash
helm test artifactory -n tools
```
![pods](/images/12.png) 

Login and configure artifactory. Get a free trial licence [here](https://jfrog.com/start-free/)

![pods](/images/13.png)

