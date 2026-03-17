# Kubernetes Deployment Guide for LoxiLB Monitoring Stack

This guide explains how to deploy the Prometheus and Grafana monitoring stack to Kubernetes, equivalent to the Docker Compose setup.

## Overview

The Kubernetes deployment includes:
- **Prometheus** - Metrics collection and storage
- **Grafana** - Visualization and dashboards
- **Persistent storage** - For both Prometheus and Grafana data
- **ConfigMaps** - For configuration files and dashboards

## Prerequisites

- Kubernetes cluster (1.19+)
- `kubectl` configured to access your cluster
- Sufficient storage provisioner for PersistentVolumeClaims
- Access to the LoxiLB Enterprise endpoint

## Architecture

```
monitoring namespace
├── Prometheus (Deployment + Service)
│   ├── ConfigMap: prometheus-config
│   └── PVC: prometheus-data (10Gi)
└── Grafana (Deployment + Service)
    ├── ConfigMap: grafana-datasources
    ├── ConfigMap: grafana-dashboards-config
    ├── ConfigMap: grafana-dashboard-loxilb          (grafana-loxilb-dashboard.json, 53 panels)
    ├── ConfigMap: grafana-dashboard-simple2         (grafana-dashboards-simple2.json, 33 panels)
    ├── ConfigMap: grafana-dashboard-sockproxy-ai    (sockproxy-ai-workload-dashboard.json, 17 panels)
    ├── ConfigMap: grafana-dashboard-sockproxy-conn  (sockproxy-connections-dashboard.json, 20 panels)
    ├── ConfigMap: grafana-dashboard-ai-gateway      (loxilb-ai-gateway-dashboard.json, 21 panels) ⭐ NEW
    ├── ConfigMap: grafana-dashboard-observability   (loxilb-observability-dashboard.json, 12 panels) ⭐ NEW
    └── PVC: grafana-data (5Gi)
```

## Deployment Steps

### 1. Create the Dashboard ConfigMaps

The dashboard ConfigMap (`k8s/02-dashboard-configmap.yaml`) already contains all 6 dashboards. To regenerate it from scratch:

```bash
# Generate the consolidated dashboard ConfigMap from all JSON files
kubectl create configmap grafana-dashboard-loxilb \
  --from-file=loxilb.json=grafana-loxilb-dashboard.json \
  --from-file=simple2.json=grafana-dashboards-simple2.json \
  --from-file=sockproxy-ai.json=sockproxy-ai-workload-dashboard.json \
  --from-file=sockproxy-conn.json=sockproxy-connections-dashboard.json \
  --from-file=ai-gateway.json=loxilb-ai-gateway-dashboard.json \
  --from-file=observability.json=loxilb-observability-dashboard.json \
  --namespace=monitoring \
  --dry-run=client -o yaml > k8s/02-dashboard-configmap.yaml
```

> **Note**: The pre-built `k8s/02-dashboard-configmap.yaml` already includes all 6 dashboards (156+ panels). Only regenerate if you have modified dashboard JSON files.

### 2. Apply All Manifests

Deploy all resources in order:

```bash
# Create namespace
kubectl apply -f k8s/00-namespace.yaml

# Create ConfigMaps
kubectl apply -f k8s/01-configmaps.yaml
kubectl apply -f k8s/02-dashboard-configmap.yaml

# Create PersistentVolumeClaims
kubectl apply -f k8s/03-pvcs.yaml

# Deploy Prometheus
kubectl apply -f k8s/04-prometheus-deployment.yaml

# Deploy Grafana
kubectl apply -f k8s/05-grafana-deployment.yaml
```

Or apply all at once:

```bash
kubectl apply -f k8s/
```

### 3. Verify Deployment

Check that all pods are running:

```bash
kubectl get pods -n monitoring
```

Expected output:
```
NAME                          READY   STATUS    RESTARTS   AGE
prometheus-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
grafana-xxxxxxxxxx-xxxxx      1/1     Running   0          2m
```

Check services:

```bash
kubectl get svc -n monitoring
```

Check PVCs:

```bash
kubectl get pvc -n monitoring
```

## Accessing the Services

### Option 1: Port Forwarding (Development/Testing)

**Prometheus:**
```bash
kubectl port-forward -n monitoring svc/prometheus 9090:9090
```
Access at: http://localhost:9090

**Grafana:**
```bash
kubectl port-forward -n monitoring svc/grafana 3000:3000
```
Access at: http://localhost:3000
- Username: `admin`
- Password: `admin`

### Option 2: NodePort Service (On-premise)

Edit the service type to `NodePort`:

```bash
kubectl patch svc grafana -n monitoring -p '{"spec":{"type":"NodePort"}}'
kubectl patch svc prometheus -n monitoring -p '{"spec":{"type":"NodePort"}}'
```

Get the NodePort:
```bash
kubectl get svc -n monitoring
```

### Option 3: LoadBalancer (Cloud)

Edit the service type to `LoadBalancer`:

```bash
kubectl patch svc grafana -n monitoring -p '{"spec":{"type":"LoadBalancer"}}'
kubectl patch svc prometheus -n monitoring -p '{"spec":{"type":"LoadBalancer"}}'
```

Get the external IP:
```bash
kubectl get svc -n monitoring
```

### Option 4: Ingress (Production)

Create an Ingress resource (example):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: monitoring-ingress
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: grafana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000
  - host: prometheus.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus
            port:
              number: 9090
```

## Configuration

### Update LoxiLB Endpoint

Edit the Prometheus ConfigMap to update your LoxiLB Enterprise endpoint:

```bash
kubectl edit configmap prometheus-config -n monitoring
```

Update the target IP address in the `static_configs` section.

Then restart Prometheus:

```bash
kubectl rollout restart deployment/prometheus -n monitoring
```

### Adjust Storage Size

Edit PVC specifications in `k8s/03-pvcs.yaml` before applying:
- Prometheus: Default 10Gi (modify based on retention and scrape frequency)
- Grafana: Default 5Gi (modify based on dashboard and user data)

### Storage Class

**Option 1: Using Dynamic Provisioning (Cloud or with StorageClass)**

If your cluster requires a specific storage class, uncomment and set it in `k8s/03-pvcs.yaml`:

```yaml
storageClassName: your-storage-class
```

**Option 2: Using hostPath (Local/Development)**

If you don't have a storage class configured, use the hostPath variant:

```bash
# Use the hostPath version instead of the regular PVC file
kubectl apply -f k8s/03-pvcs-hostpath.yaml
```

This will create PersistentVolumes using hostPath at `/mnt/data/prometheus` and `/mnt/data/grafana` on the node.

**Note:** hostPath volumes are tied to a specific node and should only be used for:
- Single-node clusters (minikube, kind, Docker Desktop)
- Development/testing environments
- When you want persistent data on a specific node

For production multi-node clusters, use a proper StorageClass with dynamic provisioning.

### Grafana Admin Password

Change the admin password by updating the environment variable in `k8s/05-grafana-deployment.yaml`:

```yaml
- name: GF_SECURITY_ADMIN_PASSWORD
  value: "your-secure-password"
```

Or use a Secret:

```bash
kubectl create secret generic grafana-admin \
  --from-literal=password=your-secure-password \
  -n monitoring
```

Then update the deployment to use the secret.

## Monitoring & Maintenance

### View Logs

```bash
# Prometheus logs
kubectl logs -f -n monitoring deployment/prometheus

# Grafana logs
kubectl logs -f -n monitoring deployment/grafana
```

### Scale (if needed)

```bash
# Not recommended for Prometheus (stateful)
kubectl scale deployment/grafana -n monitoring --replicas=2
```

### Backup Data

Backup the persistent volumes containing:
- Prometheus data: `/prometheus`
- Grafana data: `/var/lib/grafana`

### Update Configuration

After modifying ConfigMaps, restart the deployments:

```bash
kubectl rollout restart deployment/prometheus -n monitoring
kubectl rollout restart deployment/grafana -n monitoring
```

## Troubleshooting

### Pods Not Starting

```bash
kubectl describe pod <pod-name> -n monitoring
kubectl logs <pod-name> -n monitoring
```

### PVC Pending

Check if your cluster has a default storage class:

```bash
kubectl get storageclass
```

If not, you may need to:
1. Install a storage provisioner
2. Specify a storage class in the PVC manifests

### Dashboard Not Loading

1. Verify the ConfigMap was created correctly:
   ```bash
   kubectl get configmap grafana-dashboard-loxilb -n monitoring
   kubectl describe configmap grafana-dashboard-loxilb -n monitoring | grep "Data Keys"
   # Should show: loxilb.json, simple2.json, sockproxy-ai.json, sockproxy-conn.json, ai-gateway.json, observability.json
   ```

2. Check Grafana logs for provisioning errors:
   ```bash
   kubectl logs -n monitoring deployment/grafana | grep -i dashboard
   ```

### Connection Issues

Verify network connectivity:
```bash
# Test Prometheus -> LoxiLB
kubectl exec -it -n monitoring deployment/prometheus -- wget -O- http://54.180.113.121:11111/netlox/v1/metrics

# Test Grafana -> Prometheus
kubectl exec -it -n monitoring deployment/grafana -- wget -O- http://prometheus:9090/api/v1/status/config
```

## Cleanup

To remove all resources:

```bash
kubectl delete namespace monitoring
```

Or remove individually:

```bash
kubectl delete -f k8s/
```

## Differences from Docker Compose

| Feature | Docker Compose | Kubernetes |
|---------|---------------|------------|
| Networking | Bridge network | ClusterIP Services |
| Storage | Named volumes | PersistentVolumeClaims |
| Configuration | Volume mounts | ConfigMaps |
| Restart Policy | `unless-stopped` | Deployment controller |
| Service Discovery | Container names | DNS (service.namespace) |
| Scaling | Manual | Declarative (replicas) |

## Production Recommendations

1. **Security:**
   - Use Secrets for passwords instead of environment variables
   - Enable TLS/SSL with Ingress
   - Implement network policies
   - Use RBAC for service accounts

2. **High Availability:**
   - Consider Thanos for Prometheus HA
   - Use ReadWriteMany PVCs for Grafana if scaling

3. **Monitoring:**
   - Set up alerts for pod health
   - Monitor PVC usage
   - Configure resource limits appropriately

4. **Backup:**
   - Implement automated PV backups
   - Export Grafana dashboards regularly
   - Backup Prometheus data or use remote write

## Additional Resources

- [Prometheus Kubernetes Guide](https://prometheus.io/docs/prometheus/latest/installation/)
- [Grafana Kubernetes Guide](https://grafana.com/docs/grafana/latest/setup-grafana/installation/kubernetes/)
- [Kubernetes Storage Documentation](https://kubernetes.io/docs/concepts/storage/)
