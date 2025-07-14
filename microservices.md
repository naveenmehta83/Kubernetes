Alright, let's build a production-grade stateless microservices application on Kubernetes from the ground up! This guide will cover the application code, Dockerfiles, and all necessary Kubernetes YAML manifests, along with deployment and verification steps.

Our application will consist of three stateless microservices:

1.  **`db-mock`**: A simple backend service that simulates a database, returning a fixed "data" message.
2.  **`api-service`**: An intermediate API that calls the `db-mock` service and then returns a combined message.
3.  **`frontend`**: A simple web server that serves an HTML page. This page will contain JavaScript that makes an HTTP request to the `api-service` and displays the result.

---

### **Step 1: Application Code & Dockerfiles**

We'll use Python Flask for simplicity across all services.

#### 1.1 `db-mock` Service

**`db_mock_app.py`**
```python
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def get_db_data():
    print("DB-Mock: Received request.")
    return "Data from DB-Mock!"

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port)
```

**`Dockerfile.db`**
```dockerfile
FROM python:3.9-slim-buster
WORKDIR /app
COPY db_mock_app.py .
EXPOSE 5000
CMD ["python", "db_mock_app.py"]
```

#### 1.2 `api-service`

**`api_app.py`**
```python
from flask import Flask
import os
import requests

app = Flask(__name__)

# Retrieve DB_MOCK_URL from environment variable injected by ConfigMap
DB_MOCK_URL = os.environ.get('DB_MOCK_URL', 'http://localhost:5000') # Fallback for local testing

@app.route('/api')
def get_api_data():
    print(f"API-Service: Received request. Calling DB-Mock at {DB_MOCK_URL}")
    try:
        response = requests.get(DB_MOCK_URL, timeout=2)
        db_data = response.text
        return f"API response with: {db_data}"
    except requests.exceptions.RequestException as e:
        print(f"API-Service: Error calling DB-Mock: {e}")
        return f"API response: Error fetching data from DB-Mock: {e}", 500

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5001))
    app.run(host='0.0.0.0', port=port)
```

**`Dockerfile.api`**
```dockerfile
FROM python:3.9-slim-buster
WORKDIR /app
COPY api_app.py .
RUN pip install requests flask # Install dependencies
EXPOSE 5001
CMD ["python", "api_app.py"]
```

#### 1.3 `frontend` Service

**`frontend_app.py`**
```python
from flask import Flask, render_template_string
import os

app = Flask(__name__)

# Retrieve API_URL from environment variable injected by ConfigMap
API_URL = os.environ.get('API_URL', 'http://localhost:5001') # Fallback for local testing

# Simple HTML template for the frontend
HTML_TEMPLATE = """
<!DOCTYPE html>
<html>
<head>
    <title>Microservice Frontend</title>
    <style>
        body { font-family: sans-serif; text-align: center; margin-top: 50px; }
        #result { margin-top: 20px; font-size: 1.2em; color: #333; }
    </style>
</head>
<body>
    <h1>Welcome to My Microservice App!</h1>
    <button onclick="fetchData()">Fetch Data from API</button>
    <p id="result">Click the button to get data from the API service.</p>

    <script>
        const API_ENDPOINT = "{{ api_url }}"; // Injected API URL

        async function fetchData() {
            document.getElementById('result').innerText = 'Fetching data...';
            try {
                const response = await fetch(API_ENDPOINT);
                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }
                const data = await response.text();
                document.getElementById('result').innerText = data;
            } catch (error) {
                console.error('Error fetching data:', error);
                document.getElementById('result').innerText = 'Failed to fetch data: ' + error.message;
            }
        }
    </script>
</body>
</html>
"""

@app.route('/')
def index():
    print(f"Frontend: Serving index page. API_URL: {API_URL}")
    return render_template_string(HTML_TEMPLATE, api_url=f"{API_URL}/api") # Pass API_URL to JS

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5002))
    app.run(host='0.0.0.0', port=port)
```

**`Dockerfile.frontend`**
```dockerfile
FROM python:3.9-slim-buster
WORKDIR /app
COPY frontend_app.py .
RUN pip install flask # Install dependencies
EXPOSE 5002
CMD ["python", "frontend_app.py"]
```

#### 1.4 Build and Push Docker Images

Replace `your-registry-username` with your Docker Hub username or your private registry path.

```bash
# For db-mock
docker build -t your-registry-username/db-mock-app:v1.0.0 -f Dockerfile.db .
docker push your-registry-username/db-mock-app:v1.0.0

# For api-service
docker build -t your-registry-username/api-service-app:v1.0.0 -f Dockerfile.api .
docker push your-registry-username/api-service-app:v1.0.0

# For frontend
docker build -t your-registry-username/frontend-app:v1.0.0 -f Dockerfile.frontend .
docker push your-registry-username/frontend-app:v1.0.0
```

---

### **Step 2: Kubernetes YAML Manifests**

Create a directory (e.g., `k8s-manifests`) and save these files in it, prefixed with numbers for logical application order.

#### 2.1 Namespace (`00-namespace.yaml`)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-microservice-app
```

#### 2.2 Secret (`01-secret.yaml`)

This demonstrates secret handling, even if our `db-mock` doesn't strictly use it.
**Note**: `echo -n 'my-secret-db-password' | base64` to get the base64 encoded value. In production, use a secret management tool.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: my-microservice-app
type: Opaque
data:
  DB_PASSWORD: bXktc2VjcmV0LWRiLXBhc3N3b3Jk # base64 encoded "my-secret-db-password"
```

#### 2.3 ConfigMap (`02-configmap.yaml`)

Stores configuration that changes between environments but isn't sensitive.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: my-microservice-app
data:
  # Internal service DNS names provided by Kubernetes
  DB_MOCK_URL: http://db-mock-service:5000
  API_URL: http://api-service:5001
```

#### 2.4 DB-Mock Deployment (`03-db-mock-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-mock-deployment
  namespace: my-microservice-app
  labels:
    app: db-mock
spec:
  replicas: 2 # 2 replicas for high availability
  selector:
    matchLabels:
      app: db-mock
  template:
    metadata:
      labels:
        app: db-mock
    spec:
      containers:
      - name: db-mock-container
        image: your-registry-username/db-mock-app:v1.0.0 # <--- Update this
        ports:
        - containerPort: 5000
        env:
        - name: PORT # Pass port to Flask app
          value: "5000"
        - name: DB_PASSWORD # Inject secret
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_PASSWORD
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
        livenessProbe: # Checks if the app is still running
          httpGet:
            path: /
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe: # Checks if the app is ready to receive traffic
          httpGet:
            path: /
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 5
```

#### 2.5 DB-Mock Service (`04-db-mock-service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-mock-service # Internal DNS name
  namespace: my-microservice-app
  labels:
    app: db-mock
spec:
  selector:
    app: db-mock # Selects pods with this label
  ports:
    - protocol: TCP
      port: 5000 # Service port
      targetPort: 5000 # Pod's container port
  type: ClusterIP # Internal to the cluster
```

#### 2.6 API Service Deployment (`05-api-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service-deployment
  namespace: my-microservice-app
  labels:
    app: api-service
spec:
  replicas: 3 # 3 replicas for higher availability and throughput
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      containers:
      - name: api-service-container
        image: your-registry-username/api-service-app:v1.0.0 # <--- Update this
        ports:
        - containerPort: 5001
        env:
        - name: PORT
          value: "5001"
        - name: DB_MOCK_URL # Inject DB-Mock URL from ConfigMap
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_MOCK_URL
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /api
            port: 5001
          initialDelaySeconds: 10 # Give app more time to start up and connect to DB-mock
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /api
            port: 5001
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
```

#### 2.7 API Service (`06-api-service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service # Internal DNS name
  namespace: my-microservice-app
  labels:
    app: api-service
spec:
  selector:
    app: api-service
  ports:
    - protocol: TCP
      port: 5001
      targetPort: 5001
  type: ClusterIP
```

#### 2.8 Frontend Deployment (`07-frontend-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  namespace: my-microservice-app
  labels:
    app: frontend
spec:
  replicas: 2 # 2 replicas for the frontend
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend-container
        image: your-registry-username/frontend-app:v1.0.0 # <--- Update this
        ports:
        - containerPort: 5002
        env:
        - name: PORT
          value: "5002"
        - name: API_URL # Inject API URL from ConfigMap
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: API_URL
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 5002
          initialDelaySeconds: 10
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /
            port: 5002
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
```

#### 2.9 Frontend Service (`08-frontend-service.yaml`)

This will expose our application to the internet.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: my-microservice-app
  labels:
    app: frontend
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80 # External port for the Load Balancer
      targetPort: 5002 # The actual port our frontend app listens on
  type: LoadBalancer # This provisions an external Load Balancer in cloud environments
  # Optional: Cloud-specific annotations for production-grade settings
  # For AWS EKS:
  # annotations:
  #   service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
  #   service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
  #   service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
  #   service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
  #   service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: "HTTP"
  #   service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/"
  #   service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
  #   service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"
  #   service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
  #   service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "2"
```

---

### **Step 3: Deployment Steps**

**Prerequisites:**

1.  **Kubernetes Cluster**: GKE, EKS, AKS, or Minikube running locally.
2.  **`kubectl` configured**: Pointing to your cluster.
3.  **Docker Images Pushed**: Ensure you've built and pushed all three Docker images to your container registry (as instructed in Step 1.4), and updated the `image:` paths in your Deployment YAMLs.

**Deployment Commands:**

1.  Navigate to the directory where you saved your YAML manifests (`k8s-manifests`).
2.  Apply all manifests in order:
    ```bash
    kubectl apply -f 00-namespace.yaml
    kubectl apply -f 01-secret.yaml
    kubectl apply -f 02-configmap.yaml
    kubectl apply -f 03-db-mock-deployment.yaml
    kubectl apply -f 04-db-mock-service.yaml
    kubectl apply -f 05-api-deployment.yaml
    kubectl apply -f 06-api-service.yaml
    kubectl apply -f 07-frontend-deployment.yaml
    kubectl apply -f 08-frontend-service.yaml
    ```
    *Alternatively, if files are numbered correctly, you can just `kubectl apply -f .` from within the directory, but applying individually helps catch errors earlier.*

---

### **Step 4: Verification**

1.  **Check Namespace and Resources:**
    ```bash
    kubectl get all -n my-microservice-app
    ```
    You should see:
    *   Pods for `db-mock-deployment`, `api-service-deployment`, `frontend-deployment` (all running).
    *   Services `db-mock-service` (ClusterIP), `api-service` (ClusterIP), `frontend-service` (LoadBalancer).
    *   Deployments for all three.
    *   The `db-credentials` Secret and `app-config` ConfigMap.

2.  **Get Frontend External IP:**
    This might take a minute or two for the LoadBalancer to provision in a cloud environment.
    ```bash
    kubectl get svc frontend-service -n my-microservice-app
    ```
    Look for the `EXTERNAL-IP`.
    *   **On Cloud (GKE/EKS/AKS)**: You'll see a public IP address (e.g., `34.X.Y.Z`).
    *   **On Minikube**: The `EXTERNAL-IP` will likely be `<pending>`. You need to get the URL via `minikube service frontend-service -n my-microservice-app --url`.

3.  **Test the Application:**

    *   **Via Web Browser:** Open your web browser and navigate to the `EXTERNAL-IP` (or Minikube URL). You should see the "Welcome to My Microservice App!" page. Click the "Fetch Data from API" button.
        *   **Expected Result**: After clicking, the text should change to "API response with: Data from DB-Mock!".

    *   **Via `curl` (for direct API access from outside):**
        *   If you have direct access to the `EXTERNAL-IP`, you can try:
            ```bash
            # This will hit the frontend, not the API directly
            curl http://<EXTERNAL-IP_OR_MINIKUBE_URL>
            ```

4.  **Check Pod Logs (Debugging):**
    If something isn't working, check the logs of your pods:
    ```bash
    kubectl logs -n my-microservice-app -l app=frontend -c frontend-container
    kubectl logs -n my-microservice-app -l app=api-service -c api-service-container
    kubectl logs -n my-microservice-app -l app=db-mock -c db-mock-container
    ```
    You should see the "Received request" messages in the `api-service` and `db-mock` logs when you hit the frontend.

---

### **Production-Grade Considerations Highlighted:**

*   **Stateless Microservices**: Each service is designed to be stateless, meaning any instance of a service can handle any request without relying on previous requests to that specific instance. This is crucial for horizontal scaling and resilience.
*   **Namespaces**: `my-microservice-app` isolates our application's resources from others in the cluster, improving organization and security.
*   **Resource Limits**: `requests` and `limits` are set for CPU and memory. This ensures your pods get the minimum resources they need and don't consume excessive resources, preventing noisy neighbor issues and cluster instability.
*   **Liveness/Readiness Probes**:
    *   **Liveness Probe**: Kubernetes uses this to know when to restart a container (e.g., if it's deadlocked).
    *   **Readiness Probe**: Kubernetes uses this to know when a container is ready to start accepting traffic. Services won't route traffic to Pods that aren't ready. This prevents traffic from being sent to an application still initializing or temporarily unhealthy.
*   **Environment Variables**: Used to inject configuration (`PORT`, `DB_MOCK_URL`, `API_URL`) into the containers, making them dynamic and environment-agnostic.
*   **Secrets**: `db-credentials` Secret demonstrates how to store sensitive data securely and inject it into pods as environment variables. Kubernetes stores secrets encrypted at rest.
*   **ConfigMaps**: `app-config` ConfigMap stores non-sensitive configuration data (internal service URLs) and injects it as environment variables. This decouples configuration from the application image.
*   **`ClusterIP` Services**: Used for internal communication between microservices (`frontend` -> `api-service`, `api-service` -> `db-mock-service`). They provide stable DNS names and load balancing within the cluster.
*   **`LoadBalancer` Service**: Exposes the `frontend` to external users. In cloud environments, this automatically provisions a highly available cloud load balancer, handling external traffic routing and scaling.
*   **Replicas**: Each deployment has multiple replicas (`db-mock: 2`, `api-service: 3`, `frontend: 2`) for high availability and load distribution. If a Pod fails, another takes its place, and traffic is spread across all healthy instances.

This comprehensive setup provides a robust foundation for your stateless microservices application in Kubernetes!