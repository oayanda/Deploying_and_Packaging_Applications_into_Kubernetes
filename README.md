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
  k get po,svc --namespace=ingress-nginx
  ```

![pods](/images/7.png)

Verify ingress Load balancer 

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

Configure TLS/SSL

Install cert manager to manager ssl certificate.

Cert-manager is a Kubernetes addon to automate the management and issuance of TLS certificates from various issuing sources.

```bash
# install the cert-manager CustomResourceDefinition resources

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.1/cert-manager.crds.yaml
```
![pods](/images/15.png)

 To install the chart with the release name my-release

```bash
## Add the Jetstack Helm repository
$ helm repo add jetstack https://charts.jetstack.io

## Install the cert-manager helm chart
$ helm install my-release --namespace cert-manager --version v1.11.1 jetstack/cert-manager --create-namespace
```

![pods](/images/16.png)

Certificate Issuer

Next, is to create an Issuer. We will use a Cluster Issuer so that it can be scoped globally.

```bash
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  namespace: "cert-manager"
  name: "letsencrypt-prod"
spec:
  acme:
    server: "https://acme-v02.api.letsencrypt.org/directory"
    email: "info@oayanda.com"
    privateKeySecretRef:
      name: "letsencrypt-prod"
    solvers:
    - selector:
        dnsZones:
          - "tooling.artifactory.oayanda.com"

      dns01:
        route53:
          region: "us-east-1"
          role: "arn:aws:iam::737237029972:role/dns-manager"
          hostedZoneID: <"Enter-Your-HostedZone-Here">
          accessKeyID: <Enter-your-access-key-here>
          secretAccessKeySecretRef:
            name: aws-secret
            key: secret-access-key
```
Make sure to update the manifest file and apply.

```bash
k apply -f cluster-issuer.yaml
```
![pods](/images/17.png)

To ensure that every created ingress also has TLS configured, we will need to update the ingress manifest with TLS specific configurations.

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    kubernetes.io/ingress.class: nginx
  name: artifactory
spec:
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
  tls:
  - hosts:
    - "tooling.artifactory.oayanda.com"
    secretName: "tooling.artifactory.oayanda.com"
```
![pods](/images/20.png)