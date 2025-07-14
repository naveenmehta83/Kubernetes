Okay, here's a Kubernetes Deployment YAML for Nginx with 5 replicas, incorporating both liveness and readiness probes.

This example assumes Nginx is serving content on its default port 80, and a simple HTTP GET request to the root path (`/`) is sufficient to determine its health.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-web-deployment
  labels:
    app: nginx-web
spec:
  replicas: 5 # As requested: 5 replicas
  selector:
    matchLabels:
      app: nginx-web
  template:
    metadata:
      labels:
        app: nginx-web
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest # Using the official Nginx image
        ports:
        - containerPort: 80 # Nginx default HTTP port
          name: http-web # Named port for easier reference
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /       # The path to check (root path for Nginx)
            port: http-web # Reference the named port
            # Alternatively, use the port number: port: 80
          initialDelaySeconds: 10 # Give Nginx 10 seconds to start up before first check
          periodSeconds: 10       # Check every 10 seconds
          timeoutSeconds: 5       # Wait up to 5 seconds for a response
          failureThreshold: 3     # If 3 consecutive checks fail, restart the container
        readinessProbe:
          httpGet:
            path: /       # The path to check
            port: http-web # Reference the named port
            # Alternatively, use the port number: port: 80
          initialDelaySeconds: 5  # Give Nginx 5 seconds to become ready before first check
          periodSeconds: 5        # Check every 5 seconds (more frequent for readiness)
          timeoutSeconds: 3       # Wait up to 3 seconds for a response
          failureThreshold: 2     # If 2 consecutive checks fail, mark as unready (remove from service endpoints)
```

---

### **Explanation of the Probes:**

#### **1. `livenessProbe` (Ensuring the Container is Running)**

*   **Purpose**: Kubernetes uses liveness probes to know when to restart a container. If the probe fails, Kubernetes assumes the container is unhealthy and restarts it. This helps to recover from deadlocks, application freezes, or other states where the application is "stuck" but not technically crashed.
*   **`httpGet`**: This type of probe performs an HTTP GET request to a specified path and port. If it gets a successful HTTP response (2xx or 3xx status code), the probe is considered successful.
    *   `path: /`: Nginx serves content from its root path, so a simple GET request to `/` is a good basic health check.
    *   `port: http-web`: References the named port defined in the `ports` section (`containerPort: 80`). Using named ports is a good practice as it makes the manifest more readable and less prone to errors if port numbers change.
*   **`initialDelaySeconds: 10`**: Nginx needs a moment to fully initialize. This tells Kubernetes to wait 10 seconds after the container starts before performing the first liveness check. If the probe runs too early, it might fail unnecessarily.
*   **`periodSeconds: 10`**: After the initial delay, Kubernetes will perform this check every 10 seconds.
*   **`timeoutSeconds: 5`**: If Nginx doesn't respond within 5 seconds, the probe is considered failed.
*   **`failureThreshold: 3`**: If the probe fails 3 consecutive times, Kubernetes will terminate the container and attempt to restart it.

#### **2. `readinessProbe` (Ensuring the Container is Ready for Traffic)**

*   **Purpose**: Kubernetes uses readiness probes to know when a container is ready to start accepting traffic. If the probe fails, Kubernetes removes the Pod's IP address from the Service's Endpoints, meaning no traffic will be routed to that Pod until it becomes ready again. This is crucial during deployments, scaling events, or when an application temporarily loses its backend connection.
*   **`httpGet`**: Similar to liveness, an HTTP GET to the root path.
*   **`initialDelaySeconds: 5`**: A slightly shorter delay here. Often, an app is "ready" faster than it's definitively "live" for the long run.
*   **`periodSeconds: 5`**: More frequent checks (every 5 seconds) are common for readiness. You want Kubernetes to quickly detect when a Pod is ready to receive traffic or when it becomes unready and should be removed from the load balancer.
*   **`timeoutSeconds: 3`**: A slightly shorter timeout than liveness, implying a quicker expectation for readiness.
*   **`failureThreshold: 2`**: If the probe fails 2 consecutive times, Kubernetes will mark the Pod as unready and stop sending traffic to it. This allows for quick isolation of unhealthy pods from the service.

### **Why use both?**

*   **Liveness** helps ensure the application is *running* and prevents it from getting stuck in an unresponsive state. If it fails, the container restarts.
*   **Readiness** helps ensure the application is *ready to serve requests*. If it fails, the Pod is temporarily removed from the Service's traffic rotation.

By implementing both, you achieve higher availability and better control over how your application handles its lifecycle within the cluster.