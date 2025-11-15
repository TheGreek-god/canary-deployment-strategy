# Canary Deployment Strategy

This project demonstrates a **Canary Deployment** strategy using **Kubernetes** with **Minikube** and the **NGINX Ingress Controller**. It features a simple **online shop** application with two versions (v1 and v2), and the traffic is split between them using a canary deployment pattern.

## Prerequisites

Before setting up this project, you need to have the following installed on your machine:

- **Minikube** (for creating a local Kubernetes cluster)
- **kubectl** (Kubernetes command-line tool)
- **Docker** (optional, if you want to build the Docker images locally)

### Install Minikube

You can download and install **Minikube** from the official website: https://minikube.sigs.k8s.io/docs/

### Install kubectl

Follow the official instructions to install **kubectl**: https://kubernetes.io/docs/tasks/tools/install-kubectl/

### Install Docker (Optional)

If you want to build the Docker images locally, you can install Docker from: https://www.docker.com/products/docker-desktop

## Setting Up the Project

### Step 1: Start Minikube

Start Minikube to create your local Kubernetes cluster.

```bash
minikube start
```
### Step 2: Enable the NGINX Ingress Controller
To enable the NGINX Ingress controller on Minikube, run the following command:
```
minikube addons enable ingress
```

This will deploy the NGINX Ingress controller in your Minikube cluster.

### Step 3: Set Up the Namespace

We’ll use the canary-deployment namespace for the application and related resources. Apply the following YAML file to create the namespace:
```
apiVersion: v1
kind: Namespace
metadata:
  name: canary-deployment
  ```

You can apply this file using:
```
kubectl apply -f namespace.yaml
```

### Step 4: Deploy the Application

Now, deploy both versions of the application (v1 and v2) in the canary-deployment namespace.

Deploy Version 1 (Stable):
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: online-shop-v1
  namespace: canary-deployment
  labels:
    app: online-shop
    version: v1
spec:
  replicas: 4
  selector:
    matchLabels:
      app: online-shop
      version: v1
  template:
    metadata:
      labels:
        app: online-shop
        version: v1
    spec:
      containers:
      - name: online-shop
        image: amitabhdevops/online_shop_without_footer
        ports:
        - containerPort: 3000
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "200m"
            memory: "256Mi"
            
```
You can apply this file using:
```
kubectl apply -f stable-version.yaml
```

Deploy Version 2 (Canary):
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: online-shop-v2
  namespace: canary-deployment
  labels:
    app: online-shop
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: online-shop
      version: v2
  template:
    metadata:
      labels:
        app: online-shop
        version: v2
    spec:
      containers:
      - name: online-shop
        image: amitabhdevops/online_shop
        ports:
        - containerPort: 3000
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "200m"
            memory: "256Mi"
```
You can apply this file using:
```
kubectl apply -f canary-version.yaml
```
### Step 5: Expose the Applications Using Services

Create a Service to expose both the versions of the application. Here’s the Service definition for the canary and stable versions.
```
apiVersion: v1
kind: Service
metadata:
  name: online-shop-service
  namespace: canary-deployment
spec:
  selector:
    app: online-shop
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30001
  type: NodePort
```
Apply the services with:
```
kubectl apply -f combined-service.yaml
```
### Step 6: Set Up Ingress Resources

Now, create the Ingress resources to manage external traffic routing.

Main Ingress for Stable Version:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: online-shop-ingress
  namespace: canary-deployment
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false" # Optional
spec:
  ingressClassName: nginx
  rules:
  - host: localhost  # Change this to your domain if using a real environment
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: online-shop-service
            port:
              number: 80
```

Ingress for Canary Version:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: online-shop-canary-ingress
  namespace: canary-deployment
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% traffic to canary
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: localhost  # Change this to your domain if using a real environment
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: online-shop-service
            port:
              number: 80
```
You can apply this file using:
```
kubectl apply -f ingress.yaml
```
### Step 7: Access the Application Locally

If you’re using Minikube, you can forward the Ingress controller to your local machine using port forwarding:
```
kubectl port-forward -n kube-system service/ingress-nginx-controller 8080:80
```

You can now access your application at:
```
http://localhost:8080
```

90% of traffic will be directed to the stable version (v1), and 10% will go to the canary version (v2), as defined in the Ingress annotations.

### Step 8: Monitor Traffic

You can check which version of the app is serving the request by checking the logs of the deployments or by observing the behavior of the application itself. You can also  adjust the weight of the routing to canary or stable versions based on your requirements.
```
# Check pods
kubectl get pods -n canary-deployment

# Check the distribution of pods
kubectl get pods -n canary-deployment --show-labels

# Check the service
kubectl describe svc online-shop-service -n canary-deployment
```
### Step 9: Clean Up (Optional)

If you want to clean up your resources, you can delete the deployments, services, and ingress resources using:
```
kubectl delete deployment online-shop-v1 online-shop-v2 -n canary-deployment
kubectl delete service online-shop-service -n canary-deployment
kubectl delete ingress online-shop-ingress online-shop-canary-ingress -n canary-deployment
kubectl delete namespace canary-deployment
```
# Conclusion

This project demonstrates how to use a canary deployment strategy to gradually roll out changes to an application using Minikube, Kubernetes, and NGINX Ingress. You can adjust the traffic weight in the Ingress configuration to control how much traffic goes to the canary version compared to the stable version.

