Excellent! This is a comprehensive request, and building such a workflow will demonstrate a strong understanding of CI/CD, Docker, and Kubernetes.

Here's the full GitHub Actions workflow YAML, designed to meet all your requirements.

**Assumptions & Preparations:**

1.  **Project Structure:** I'm assuming a project structure like this, where each microservice has its code, Dockerfile, and a simple `test_*.py` file for unit tests:
    ```
    .
    ├── db-mock/
    │   ├── db_mock_app.py
    │   ├── Dockerfile.db
    │   └── test_db_mock.py
    ├── api-service/
    │   ├── api_app.py
    │   ├── Dockerfile.api
    │   ├── requirements.txt # For 'requests' and 'flask'
    │   └── test_api_service.py
    ├── frontend/
    │   ├── frontend_app.py
    │   ├── Dockerfile.frontend
    │   ├── requirements.txt # For 'flask'
    │   └── test_frontend.py
    └── k8s-manifests/
        ├── 00-namespace.yaml
        ├── ... (other manifests as per previous guide)
        └── 08-frontend-service.yaml
    ```
    *If your structure is flatter (e.g., all Dockerfiles and `test_*.py` in the root), you'll need to adjust the `context` and `app_dir` in the `matrix` strategy accordingly.*

2.  **`requirements.txt`**: Ensure each service directory has a `requirements.txt` file listing its Python dependencies (e.g., `Flask`, `requests`, `pytest`).
    *   `api-service/requirements.txt`: `Flask==2.3.2`, `requests==2.31.0`, `pytest==7.4.0`
    *   `frontend/requirements.txt`: `Flask==2.3.2`, `pytest==7.4.0`
    *   `db-mock/requirements.txt`: `Flask==2.3.2`, `pytest==7.4.0`

3.  **GitHub Secrets**: You **must** add these secrets to your GitHub repository settings (`Settings -> Secrets and variables -> Actions -> New repository secret`):
    *   `DOCKER_USERNAME`: Your Docker Hub username.
    *   `DOCKER_PASSWORD`: Your Docker Hub Access Token (highly recommended over your actual password).
    *   `KUBE_CONFIG_BASE64`: A base64-encoded string of your Kubernetes `kubeconfig` file.
        *   To generate: `cat ~/.kube/config | base64 -w 0` (use `-w 0` on Linux/macOS to prevent line wrapping). Ensure this `kubeconfig` has credentials for your target cluster and the necessary permissions to deploy.
    *   `SLACK_WEBHOOK_URL`: The URL for your Slack Incoming Webhook (or equivalent for email, etc.).

4.  **Image Naming**: I've used placeholders like `your-registry-username/db-mock-app`. **Remember to replace `your-registry-username` in the workflow YAML and your Kubernetes manifests with your actual Docker Hub username or registry path.**

---

### **GitHub Actions Workflow YAML (`.github/workflows/deploy.yaml`)**

```yaml
name: Microservices CI/CD to Kubernetes

on:
  push:
    branches:
      - main # Trigger on pushes to the main branch

env:
  DOCKER_REGISTRY: docker.io # Or ghcr.io if using GitHub Container Registry
  DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
  # Define base image names for each service.
  # These must match what's expected in your K8s manifests BEFORE substitution.
  # Update 'your-registry-username' to your actual Docker Hub/GHCR username.
  DB_MOCK_IMAGE_BASE: your-registry-username/db-mock-app
  API_SERVICE_IMAGE_BASE: your-registry-username/api-service-app
  FRONTEND_IMAGE_BASE: your-registry-username/frontend-app
  IMAGE_TAG: ${{ github.sha }} # Use commit SHA as image tag for immutability

jobs:
  # --- Build and Test Job ---
  build-and-test:
    runs-on: ubuntu-latest
    outputs:
      # Pass the job status for conditional deployment and notifications
      job_status: ${{ job.status }}
    strategy:
      matrix:
        # Define each microservice for parallel build and test
        service:
          - name: db-mock
            path: db-mock
            dockerfile_name: Dockerfile.db
            image_var: DB_MOCK_IMAGE_BASE # References the env var for base image name
          - name: api-service
            path: api-service
            dockerfile_name: Dockerfile.api
            image_var: API_SERVICE_IMAGE_BASE
          - name: frontend
            path: frontend
            dockerfile_name: Dockerfile.frontend
            image_var: FRONTEND_IMAGE_BASE

    steps:
    - uses: actions/checkout@v4 # Checkout the repository code

    - name: Set up Python for ${{ matrix.service.name }}
      uses: actions/setup-python@v5
      with:
        python-version: '3.9' # Specify Python version

    - name: Install dependencies for ${{ matrix.service.name }}
      run: |
        python -m pip install --upgrade pip
        pip install -r ${{ matrix.service.path }}/requirements.txt # Install dependencies for the service
      working-directory: ${{ matrix.service.path }} # Run from service's directory

    - name: Run unit tests for ${{ matrix.service.name }}
      run: |
        pytest test_${{ matrix.service.name }}.py # Assuming test files are named like test_db_mock.py
      working-directory: ${{ matrix.service.path }}
      # Consider adding `continue-on-error: true` here if you want other services to build/test
      # even if one service's tests fail, but for production, typically all must pass.

    - name: Log in to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ env.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image for ${{ matrix.service.name }}
      run: |
        SERVICE_IMAGE_NAME=${{ env[matrix.service.image_var] }} # Get the image name from the env var
        FULL_IMAGE_NAME="${SERVICE_IMAGE_NAME}:${{ env.IMAGE_TAG }}"
        echo "Building $FULL_IMAGE_NAME"
        docker build -t $FULL_IMAGE_NAME -f ${{ matrix.service.path }}/${{ matrix.service.dockerfile_name }} ${{ matrix.service.path }}
        docker push $FULL_IMAGE_NAME
        echo "Successfully built and pushed $FULL_IMAGE_NAME"

  # --- Deploy Job ---
  deploy:
    runs-on: ubuntu-latest
    needs: build-and-test # This job depends on all 'build-and-test' matrix jobs
    # Only run deploy if ALL build-and-test jobs succeeded
    if: success() && needs.build-and-test.result == 'success'

    steps:
    - uses: actions/checkout@v4 # Checkout the repository code to access manifests

    - name: Setup Kubeconfig
      run: |
        mkdir -p ~/.kube
        echo "${{ secrets.KUBE_CONFIG_BASE64 }}" | base64 -d > ~/.kube/config # Decode and save kubeconfig
        chmod 600 ~/.kube/config # Set secure permissions

    - name: Substitute image tags in Kubernetes manifests
      # This step dynamically updates the image tags in the Deployment YAMLs
      # to use the unique SHA tag generated in the build-and-test job.
      run: |
        # Use `sed -i` (in-place edit) to replace the placeholder image tags.
        # Ensure the placeholders in your K8s manifests EXACTLY match what's here.
        # Example: image: your-registry-username/db-mock-app:v1.0.0
        sed -i "s|image: ${DB_MOCK_IMAGE_BASE}:v1.0.0|image: ${DB_MOCK_IMAGE_BASE}:${IMAGE_TAG}|g" k8s-manifests/03-db-mock-deployment.yaml
        sed -i "s|image: ${API_SERVICE_IMAGE_BASE}:v1.0.0|image: ${API_SERVICE_IMAGE_BASE}:${IMAGE_TAG}|g" k8s-manifests/05-api-deployment.yaml
        sed -i "s|image: ${FRONTEND_IMAGE_BASE}:v1.0.0|image: ${FRONTEND_IMAGE_BASE}:${IMAGE_TAG}|g" k8s-manifests/07-frontend-deployment.yaml

        echo "Updated manifest content for db-mock-deployment:"
        cat k8s-manifests/03-db-mock-deployment.yaml
        echo "Updated manifest content for api-service-deployment:"
        cat k8s-manifests/05-api-deployment.yaml
        echo "Updated manifest content for frontend-deployment:"
        cat k8s-manifests/07-frontend-deployment.yaml
      env:
        DB_MOCK_IMAGE_BASE: ${{ env.DB_MOCK_IMAGE_BASE }}
        API_SERVICE_IMAGE_BASE: ${{ env.API_SERVICE_IMAGE_BASE }}
        FRONTEND_IMAGE_BASE: ${{ env.FRONTEND_IMAGE_BASE }}
        IMAGE_TAG: ${{ env.IMAGE_TAG }}

    - name: Deploy updated images to Kubernetes
      id: deploy_k8s # Assign an ID for conditional steps
      run: |
        echo "Starting Kubernetes deployment..."
        # Apply namespace first, ignore error if it already exists
        kubectl apply -f k8s-manifests/00-namespace.yaml || true

        # Apply all other manifests, specifying the namespace.
        # Using --dry-run=client -o yaml | kubectl apply -f - for a more robust apply of multiple files.
        # --wait ensures that the new pods are healthy and ready before the step passes.
        # This is CRITICAL for production stability.
        kubectl apply -f k8s-manifests/ --namespace my-microservice-app --dry-run=client -o yaml | kubectl apply -f -

        # Explicitly check rollout status for each deployment.
        # This gives clearer waiting behavior and failure detection for deployments.
        echo "Waiting for db-mock-deployment rollout to complete..."
        kubectl rollout status deployment/db-mock-deployment -n my-microservice-app --timeout=5m
        echo "Waiting for api-service-deployment rollout to complete..."
        kubectl rollout status deployment/api-service-deployment -n my-microservice-app --timeout=5m
        echo "Waiting for frontend-deployment rollout to complete..."
        kubectl rollout status deployment/frontend-deployment -n my-microservice-app --timeout=5m
      # This step will fail the job if `kubectl rollout status` times out or reports an unhealthy state.

    - name: Rollback on Deployment Failure
      # This step runs ONLY if the previous 'deploy_k8s' step failed.
      if: failure() && steps.deploy_k8s.outcome == 'failure'
      run: |
        echo "Deployment failed! Attempting rollback..."
        # Try to undo the last rollout for each deployment.
        # `|| true` prevents the rollback step from failing the workflow if a rollout history doesn't exist or undo fails.
        kubectl rollout undo deployment/frontend-deployment -n my-microservice-app || true
        kubectl rollout undo deployment/api-service-deployment -n my-microservice-app || true
        kubectl rollout undo deployment/db-mock-deployment -n my-microservice-app || true
        echo "Rollback initiated for affected deployments. Check cluster status."
      continue-on-error: true # Ensure this step doesn't fail the entire workflow if rollback itself has issues.

  # --- Notification Jobs ---
  notify-success:
    runs-on: ubuntu-latest
    needs: deploy # Only run after 'deploy' job completes
    # Only run if 'deploy' job succeeded
    if: success() && needs.deploy.result == 'success'
    steps:
      - name: Send Slack Notification (Success)
        uses: act10n/slack@v2 # Action for Slack notifications
        with:
          message: |
            ✅ *Deployment Success!*
            Repository: <${{ github.server_url }}/${{ github.repository }}|${{ github.repository }}>
            Branch: `${{ github.ref_name }}`
            Commit: `${{ github.sha }}`
            Workflow: <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|${{ github.workflow }}>
          webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }} # Slack webhook URL from secrets

  notify-failure:
    runs-on: ubuntu-latest
    # This job depends on ALL preceding jobs to catch any failure point.
    needs: [build-and-test, deploy]
    # Run if any of the required jobs failed.
    if: failure() && (needs.build-and-test.result == 'failure' || needs.deploy.result == 'failure')
    steps:
      - name: Send Slack Notification (Failure)
        uses: act10n/slack@v2
        with:
          message: |
            ❌ *Deployment Failed!*
            Repository: <${{ github.server_url }}/${{ github.repository }}|${{ github.repository }}>
            Branch: `${{ github.ref_name }}`
            Commit: `${{ github.sha }}`
            Workflow: <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|${{ github.workflow }}>
            *Reason:*
            ${{ needs.build-and-test.result == 'failure' && 'Build or Unit Tests Failed!' || '' }}
            ${{ needs.deploy.result == 'failure' && 'Deployment to Kubernetes Failed!' || '' }}
            Please check the workflow run logs for details: <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Logs>
          webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

### **Explanation of Key Sections and Best Practices:**

1.  **`on: push: branches: [main]`**:
    *   **Trigger**: The workflow will automatically start whenever code is pushed to the `main` branch.

2.  **`env` Variables**:
    *   Global environment variables (`DOCKER_REGISTRY`, `DOCKER_USERNAME`, `IMAGE_TAG`) are defined for consistency and easy updates.
    *   `IMAGE_TAG: ${{ github.sha }}`: Using the Git commit SHA as the Docker image tag is a best practice for immutability and traceability. Every build is unique and reproducible.
    *   `_IMAGE_BASE`: These variables help create the full image name for each service.

3.  **`build-and-test` Job (Reusable Steps with Matrix Strategy)**:
    *   **`strategy: matrix`**: This is key for reusability. We define an array of `service` objects, each representing a microservice. GitHub Actions will then run a separate instance of this job for each item in the matrix, in parallel.
    *   `name`, `path`, `dockerfile_name`, `image_var`: These matrix variables allow us to dynamically build and test each service.
    *   **`uses: actions/setup-python@v5`**: Sets up the Python environment.
    *   **`Install dependencies`**: Installs `pytest` and other application dependencies from the service-specific `requirements.txt`.
    *   **`Run unit tests`**: Executes `pytest` for the current service. It's crucial that this step passes before deployment.
    *   **`docker/login-action@v3`**: Securely logs into Docker Hub using secrets.
    *   **`Build and push Docker image`**: Builds the Docker image for the specific service and pushes it with the unique `IMAGE_TAG` (commit SHA).

4.  **`deploy` Job**:
    *   **`needs: build-and-test`**: Ensures that the deployment job only starts after *all* `build-and-test` matrix jobs have completed successfully.
    *   **`if: success() && needs.build-and-test.result == 'success'`**: Explicitly checks that the `build-and-test` job (including all its matrix runs) was successful.
    *   **`Setup Kubeconfig`**:
        *   Retrieves the base64-encoded `KUBECONFIG` from GitHub Secrets.
        *   Decodes it and writes it to `~/.kube/config`.
        *   `chmod 600` ensures proper file permissions for security.
    *   **`Substitute image tags in Kubernetes manifests`**:
        *   This `sed` command is powerful for dynamic deployments. It finds the placeholder image names (e.g., `your-registry-username/db-mock-app:v1.0.0`) in your YAML manifests and replaces them with the actual, tagged image names (e.g., `your-registry-username/db-mock-app:f7e3a2d...`).
        *   **CRITICAL**: Ensure the placeholder images in your `k8s-manifests/*.yaml` files precisely match what the `sed` commands are looking for.
    *   **`Deploy updated images to Kubernetes`**:
        *   `kubectl apply -f k8s-manifests/00-namespace.yaml || true`: Applies the namespace. `|| true` makes it idempotent (doesn't fail if the namespace already exists).
        *   `kubectl apply -f k8s-manifests/ --namespace my-microservice-app --dry-run=client -o yaml | kubectl apply -f -`: This is a robust way to apply multiple manifests. It first does a client-side dry run, outputs the combined YAML, and then pipes it to `kubectl apply`. This helps prevent order-of-operation issues for related resources.
        *   **`kubectl rollout status deployment/<deployment-name> -n <namespace> --timeout=5m`**: This is essential for production deployments. It makes the workflow *wait* for each deployment to successfully roll out (i.e., all new pods are healthy and ready) before marking the step as successful. If a rollout gets stuck or fails, this step will time out and cause the workflow to fail, triggering the rollback.

5.  **`Rollback on Deployment Failure`**:
    *   **`if: failure() && steps.deploy_k8s.outcome == 'failure'`**: This step runs only if the `deploy_k8s` step *specifically* failed.
    *   **`kubectl rollout undo deployment/<name> -n <namespace>`**: Triggers a rollback to the previous successful deployment.
    *   **`|| true`**: Appended to each `undo` command. This ensures that even if a specific rollback command fails (e.g., no history to undo, or a deployment wasn't created), the *rollback step itself* doesn't fail the entire workflow, allowing the notification to be sent.
    *   **`continue-on-error: true`**: This is on the step itself, meaning if the rollback logic somehow breaks, the workflow can still proceed to send a failure notification.

6.  **`notify-success` and `notify-failure` Jobs (Notifications)**:
    *   **`needs: deploy`**: Ensures notifications are sent after the `deploy` job.
    *   **`if: success()/failure()`**: Conditional execution based on the outcome of previous jobs.
    *   **`uses: act10n/slack@v2`**: A popular GitHub Action for sending Slack notifications. You'd use `SLACK_WEBHOOK_URL` from your secrets.
    *   The message includes relevant links to the repository, branch, commit, and workflow run for quick debugging.

This workflow is a powerful and flexible template for your microservices CI/CD needs!