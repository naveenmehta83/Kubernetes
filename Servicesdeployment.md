Alright, aspiring Kubernetes Master! This is a fantastic exercise for senior roles. It covers core concepts like deployments, services, inter-service communication, and external exposure. Let's design this production-like setup.

We'll break this down into clear steps:

1.  **Backend Application**: A simple Python Flask app that returns "Hello from backend!". We need to containerize this.
2.  **Backend Deployment**: 5 replicas of our backend application.
3.  **Backend Service (ClusterIP)**: To allow the frontend to communicate with the backend internally.
4.  **Frontend Nginx Configuration**: To proxy requests from the internet to the backend.
5.  **Frontend Deployment**: 3 replicas of Nginx, configured to use our custom configuration.
6.  **Frontend Service (LoadBalancer)**: To expose the Nginx frontend to external users.
7.  **Deployment Instructions**: How to apply these manifests and test.

---

### **Step 1: Backend Application and Dockerfile**

First, let's create our simple backend application.

**`backend_app.py`**
```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_backend():
    return "Hello from backend!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**`Dockerfile` (for backend)**
```dockerfile
# Use a slim Python base image
FROM python:3.9-slim-buster

# Set the working directory in the container
WORKDIR /app

# Copy the requirements file and install dependencies (if any, though none for this simple app)
# COPY requirements.txt .
# RUN pip install -r requirements.txt

# Copy the application code into the container
COPY backend_app.py .

# Expose the port the app runs on
EXPOSE 5000

# Command to run the application
CMD ["python", "backend_app.py"]
```

**Instructions for Building & Pushing Backend Image:**
You would typically build and push this to a container registry (like Docker Hub, Google Container Registry, Azure Container Registry, or AWS ECR).
```bash
# 1. Build the Docker image
docker build -t your-registry-username/backend-app:v1.0.0 .

# 2. Log in to your Docker registry (e.g., Docker Hub)
# docker login

# 3. Push the image to your registry
docker push your-registry-username/backend-app:v1.0.0
```
**Important:** Replace `your-registry-username/backend-app:v1.0.0` with your actual image path.

---

### **Step 2: Backend Deployment Manifest (`backend-deployment.yaml`)**

This defines our backend application Pods and their desired state.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  labels:
    app: backend
spec:
  replicas: 5 # As requested
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend-container
        image: your-registry-username/backend-app:v1.0.0 # <--- IMPORTANT: Replace with your pushed image!
        ports:
        - containerPort: 5000 # The port our Flask app listens on
          name: http
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

---

### **Step 3: Backend Service Manifest (`backend-service.yaml`)**

This `ClusterIP` Service exposes the backend internally, allowing the frontend to reach it via a stable DNS name.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service # This will be the DNS name for frontend to access backend
  labels:
    app: backend
spec:
  selector:
    app: backend # Matches the labels on our backend Pods
  ports:
    - protocol: TCP
      port: 80 # The port the Service will be exposed on internally
      targetPort: http # References the named port 'http' (5000) in the backend Deployment
  type: ClusterIP # Default, but good to be explicit
```
*Explanation*: Frontend pods can now make HTTP requests to `http://backend-service:80` (or just `http://backend-service` if the default port is assumed), and Kubernetes will route that traffic to one of the healthy backend Pods on port `5000`.

---

### **Step 4: Frontend Nginx Configuration (`nginx-configmap.yaml`)**

Nginx needs to know how to serve requests and how to proxy to our backend. We'll use a `ConfigMap` to inject this configuration.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {
        worker_connections 1024;
    }

    http {
        include /etc/nginx/mime.types; # Required for static file serving

        server {
            listen 80; # Nginx listens on port 80

            # Serve static files from Nginx default directory
            location / {
                root /usr/share/nginx/html;
                index index.html index.htm;
            }

            # Proxy requests starting with /api to the backend service
            location /api {
                proxy_pass http://backend-service:80; # Communicate with our backend-service
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
            }
        }
    }
```
*Explanation*:
*   `listen 80;`: Nginx listens for incoming HTTP requests on port 80.
*   `location /`: This handles requests to the root path, serving any static content you might place in `/usr/share/nginx/html` inside the Nginx container.
*   `location /api`: This is crucial! Any request to `http://your-frontend-ip/api` will be proxied to our internal `backend-service` on port 80 (which then forwards to the backend Pods on port 5000).

---

### **Step 5: Frontend Deployment Manifest (`frontend-deployment.yaml`)**

This deploys 3 Nginx replicas and mounts our custom configuration.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  labels:
    app: frontend
spec:
  replicas: 3 # As requested
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx-frontend
        image: nginx:latest # Using the official Nginx image
        ports:
        - containerPort: 80 # Nginx listens on port 80
          name: http
        volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf # Mounts only the nginx.conf file from the ConfigMap
      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config # References the ConfigMap we created earlier
          items:
            - key: nginx.conf
              path: nginx.conf
```

---

### **Step 6: Frontend Service Manifest (`frontend-service.yaml`)**

This `LoadBalancer` Service exposes the Nginx frontend to external users. In a cloud environment, this will provision a cloud load balancer.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  labels:
    app: frontend
spec:
  selector:
    app: frontend # Matches the labels on our frontend Pods
  ports:
    - protocol: TCP
      port: 80 # The port external users will hit on the LoadBalancer
      targetPort: http # References the named port 'http' (80) in the frontend Deployment
  type: LoadBalancer # This is key for external exposure in cloud environments
  # For cloud-specific annotations, you might add:
  # annotations:
  #   service.beta.kubernetes.io/aws-load-balancer-type: nlb # Example for AWS
  #   service.beta.kubernetes.io/azure-load-balancer-resource-group: my-rg # Example for Azure
```
*Explanation*: When you apply this, your cloud provider (AWS, GCP, Azure, etc.) will create an external Load Balancer that forwards traffic on port 80 to your frontend Pods.

---

### **Step 7: Deployment Instructions**

Now, let's bring it all together.

**Prerequisites:**

1.  **Kubernetes Cluster:** You need a running Kubernetes cluster (e.g., `minikube` for local testing, or a cloud-managed cluster like EKS, GKE, AKS).
2.  **`kubectl` configured:** Ensure your `kubectl` is configured to connect to your cluster.
3.  **Backend Image Pushed:** Make sure you've built and pushed your `your-registry-username/backend-app:v1.0.0` image to a public or accessible private registry as instructed in Step 1.

**Deployment Steps:**

1.  **Save the Manifests:**
    Save the YAML content for each component into separate files:
    *   `backend-deployment.yaml`
    *   `backend-service.yaml`
    *   `nginx-configmap.yaml`
    *   `frontend-deployment.yaml`
    *   `frontend-service.yaml`

2.  **Apply the Manifests:**
    Apply them in a logical order. The backend should be ready before the frontend tries to connect.
    ```bash
    kubectl apply -f backend-deployment.yaml
    kubectl apply -f backend-service.yaml
    kubectl apply -f nginx-configmap.yaml
    kubectl apply -f frontend-deployment.yaml
    kubectl apply -f frontend-service.yaml
    ```

3.  **Verify Deployments and Pods:**
    ```bash
    kubectl get deployments
    kubectl get pods -o wide # Check if all pods are running and on which nodes
    ```
    You should see `backend-deployment` with 5 replicas and `frontend-deployment` with 3 replicas.

4.  **Verify Services:**
    ```bash
    kubectl get services
    ```
    Look for `backend-service` (should have a `ClusterIP`) and `frontend-service` (should have an `EXTERNAL-IP` if you're on a cloud provider, or a `NodePort` if running locally and using `minikube service frontend-service`).

    *   If using a cloud provider, wait a minute or two for the `EXTERNAL-IP` of `frontend-service` to provision.
    *   If using `minikube`:
        ```bash
        minikube service frontend-service --url
        ```
        This command will give you the URL to access your frontend.

5.  **Test the Application:**

    *   **Access the Frontend (which proxies to backend):**
        Once you have the `EXTERNAL-IP` (or `minikube` URL) for `frontend-service`, open your web browser or use `curl`.
        ```bash
        # Replace <EXTERNAL-IP> with the actual IP/URL from 'kubectl get services' or 'minikube service'
        curl http://<EXTERNAL-IP>/api
        ```
        **Expected Output:** `Hello from backend!`

    *   **Test Internal Backend Access (Optional, for debugging):**
        You can exec into a frontend pod and try to curl the backend service directly to verify internal communication.
        ```bash
        # Get the name of one of your frontend pods
        FRONTEND_POD_NAME=$(kubectl get pods -l app=frontend -o jsonpath='{.items[0].metadata.name}')

        # Exec into the pod and try to curl the backend service
        kubectl exec -it $FRONTEND_POD_NAME -- curl http://backend-service:80
        ```
        **Expected Output:** `Hello from backend!`

---

This setup effectively demonstrates:
*   **Deployment Scaling**: Managing multiple replicas for both frontend and backend.
*   **Service Discovery**: Frontend automatically finds the backend via `backend-service` DNS.
*   **Internal Communication**: `ClusterIP` for inter-service communication within the cluster.
*   **External Exposure**: `LoadBalancer` for exposing the application to the internet.
*   **Nginx as a Reverse Proxy**: Using `ConfigMap` to manage Nginx configuration for advanced routing.
*   **Containerization**: Packaging your application for Kubernetes.

This comprehensive example should give you plenty to discuss and demonstrate your understanding in a senior-level interview! Good luck!