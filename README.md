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