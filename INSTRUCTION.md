### How to deploy the app to k8s

```bash
kubectl apply -f .infrastructure/namespace.yml 
kubectl apply -f .infrastructure/deployment.yml 
kubectl apply -f .infrastructure/hpa.yml 
kubectl apply -f .infrastructure/nodeport.yml 
```

### Configuration & Strategy Justifications
```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "250m"
```
Memory Requests (128Mi) & Limits (256Mi):

Why: Django WSGI/ASGI applications with framework dependencies (REST framework, ORM models) require approximately 70–100Mi of RAM during startup and baseline execution. A lower memory request (such as 30Mi) leads to container crashes (OOMKilled) during application initialization.

Limits: A ceiling of 256Mi prevents memory leaks from affecting neighboring pods on the node while accommodating transient spikes during heavier HTTP payload processing.

CPU Requests (100m) & Limits (250m):

Why: 100m (0.1 vCPU) guarantees enough CPU cycles for the Python interpreter to handle incoming connections reliably under baseline usage. 250m allows short bursts for task processing without throttled performance degradation.

### Deployment Strategy Configuration

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```
maxUnavailable: 1:

Why: With a base capacity of 2 replicas, allowing a maximum of 1 pod to be unavailable guarantees that at least 50% of the serving capacity remains online during updates, preventing downtime.

maxSurge: 1:

Why: This configuration allows Kubernetes to create 1 extra temporary pod (totaling 3 pods during deployment rollout) to ensure new code passes health/readiness checks before old pods are terminated, realizing a zero-downtime deployment.

### Horizontal Pod Autoscaler (HPA) Choice

```yaml
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: todoapp
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
```
minReplicas: 2 / maxReplicas: 5:

Maintaining a minimum of 2 replicas preserves high availability across node boundaries or pod restarts. Scaling up to 5 replicas prevents resource exhaustion under elevated load.

Target CPU and Memory Utilization (70%):

Why: A threshold of 70% leaves a 30% safety buffer for pods to handle ongoing traffic spikes while Kubernetes provisions and warms up new pod replicas (mitigating latency spikes during rapid scaling events).

### How to access the app after deployment

Access the application health endpoint directly using the allocated NodePort (30080):

```bash
curl http://localhost:30080
```

Via Port Forwarding:
```bash
kubectl port-forward svc/todoapp 8080:80 -n todoapp
```

Then test locally:

```bash
curl http://localhost:8080
```
