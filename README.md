# AKS with Ingress Controller

## Create a resource group

1. Go to Azure Portal
2. Create a resource group - **aks-with-ingress** in your subscription

## Create an AKS Cluster

1. From Azure Portal
2. Create an Azure Kubernetes Service (AKS)

![Image.png](https://res.craft.do/user/full/93958d6f-1340-6147-fc3c-1be83d5bfef9/doc/38F148F4-36A5-4092-97B8-CBB225B36FB5/31703573-BCEB-4D0F-88F3-A7EC439F553E_2/qfSgWWpstY6q8oZCOATBqbld1CnwjxWo2uetwp1Wt9oz/Image.png)

3. Fill the details:
   - **Subscription** - Choose your subscription
   - **Resource Group -** aks-with-ingress
   - **Cluster preset configuration** - Standard
   - **Kubernetes cluster name** - choose cluster name
   - **Region** - choose a region ( same as resource group )
   - **Availability Zones** - Zone 1
   - **AKS pricing tier** - Standard
   - **Kubernetes version** - Default
   - **Node size** - Standard DS2 v2
   - **Scale Method** - manual
   - **Node count** - 1
   - **Authentication and Authorization** - Local accounts with Kubernetes RBAC
   - **Networking:**
      - **Network configuration** - Azure CNI
      - **Virtual network** - New - leave as default
   - **Integrations**:
      - **Container registry** - Create new
         - **Registry name** - choose a unique registry name ( lower case without dashes )
         - **Resource group** - Same as the cluster
         - **Region** - Same as the cluster
         - **Admin user** - Enable
         - **SKU** - Standard
1. Review + create
2. Create
3. Connect to the cluster:

```bash
az aks get-credentials --resource-group aks-with-ingress --name aks-with-ingress
```

## Install Helm

```bash
https://helm.sh/docs/intro/install/
```

## Create an ingress controller in Azure Kubernetes Service ( AKS )

To create a basic NGINX ingress controller without customizing the defaults, you'll use Helm. The following configuration uses the default configuration for simplicity. You can add parameters for customizing the deployment, like `--set controller.replicaCount=3`.

```bash
NAMESPACE=ingress-basic

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --create-namespace \
  --namespace $NAMESPACE \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

Check the load balancer service:

```bash
kubectl get services --namespace ingress-basic -o wide -w ingress-nginx-controller
```

## Run demo applications

1. Create an `aks-helloworld-one.yaml` file and copy in the following example YAML:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aks-helloworld-one  
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
          value: "Welcome to Azure Kubernetes Service (AKS)"
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

1. Create an `aks-helloworld-two.yaml` file and copy in the following example YAML:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aks-helloworld-two  
spec:
  replicas: 1
  selector:
    matchLabels:
      app: aks-helloworld-two
  template:
    metadata:
      labels:
        app: aks-helloworld-two
    spec:
      containers:
      - name: aks-helloworld-two
        image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
        ports:
        - containerPort: 80
        env:
        - name: TITLE
          value: "AKS Ingress Demo"
---
apiVersion: v1
kind: Service
metadata:
  name: aks-helloworld-two  
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: aks-helloworld-two
```

3. Run the two demo applications using `kubectl apply`:

```bash
kubectl apply -f aks-helloworld-one.yaml --namespace ingress-basic
kubectl apply -f aks-helloworld-two.yaml --namespace ingress-basic
```

4. Create an ingress route

Both applications are now running on your Kubernetes cluster. To route traffic to each application, create a Kubernetes ingress resource. The ingress resource configures the rules that route traffic to one of the two applications.

In the following example, traffic to *EXTERNAL_IP/hello-world-one* is routed to the service named `aks-helloworld-one`. Traffic to *EXTERNAL_IP/hello-world-two* is routed to the `aks-helloworld-two` service. Traffic to *EXTERNAL_IP/static* is routed to the service named `aks-helloworld-one` for static assets.

- Create a file named `hello-world-ingress.yaml` and copy in the following example YAML:

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
 - http:
      paths:
      - path: /hello-world-one(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: aks-helloworld-one
            port:
              number: 80
      - path: /hello-world-two(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: aks-helloworld-two
            port:
              number: 80
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: aks-helloworld-one
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress-static
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/rewrite-target: /static/$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /static(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: aks-helloworld-one
            port: 
              number: 80
```

- Create the ingress resource using the `kubectl apply` command.

```bash
kubectl apply -f hello-world-ingress.yaml --namespace ingress-basic
```

## Test the ingress controller

To test the routes for the ingress controller, browse to the two applications. Open a web browser to the IP address of your NGINX ingress controller, such as *EXTERNAL_IP*. The first demo application is displayed in the web browser, as shown in the following example:

![Image.png](https://res.craft.do/user/full/93958d6f-1340-6147-fc3c-1be83d5bfef9/doc/38F148F4-36A5-4092-97B8-CBB225B36FB5/301848DB-0018-437A-B13D-CFE8777D6DD4_2/kEir6ReZJQspJasnil2DsC8r8ESKuXn1xPyDPyxOW3oz/Image.png)

Now add the */hello-world-two* path to the IP address, such as *EXTERNAL_IP/hello-world-two*. The second demo application with the custom title is displayed:

![Image.png](https://res.craft.do/user/full/93958d6f-1340-6147-fc3c-1be83d5bfef9/doc/38F148F4-36A5-4092-97B8-CBB225B36FB5/7C6EFFD3-E7F7-4BB7-A399-B3EFFFAF7703_2/HIpgerPrNBJcWvBFe5gmLlhyaswexwZavR5qqNpEZoYz/Image.png)

[Create an ingress controller in Azure Kubernetes Service (AKS) - Azure Kubernetes Service](https://learn.microsoft.com/en-us/azure/aks/ingress-basic?tabs=azure-cli)

