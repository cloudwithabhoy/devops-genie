# Kubernetes Cheatsheet

> Quick-reference kubectl commands and K8s concepts.

## Core Commands / Concepts

### Cluster & Context
```bash
kubectl cluster-info
kubectl config get-contexts
kubectl config use-context <name>
kubectl get nodes
```

### Pods & Workloads
```bash
kubectl get pods -n <namespace>
kubectl get pods -A                          # all namespaces
kubectl describe pod <name>
kubectl logs <pod> -f                        # follow logs
kubectl exec -it <pod> -- bash
kubectl run tmp --image=busybox --rm -it -- sh   # debug pod
```

### Deployments
```bash
kubectl get deployments
kubectl apply -f deployment.yaml
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>
kubectl scale deployment <name> --replicas=3
kubectl set image deployment/<name> app=myapp:2.0
```

### Services & Ingress
```bash
kubectl get svc
kubectl expose deployment <name> --port=80 --type=ClusterIP
kubectl get ingress
```

### Config & Secrets
```bash
kubectl create configmap myconfig --from-file=config.yaml
kubectl create secret generic mysecret --from-literal=password=abc123
kubectl get configmaps
kubectl get secrets
```

### Namespaces & RBAC
```bash
kubectl get namespaces
kubectl create namespace dev
kubectl auth can-i create pods --as=user1
```

### Troubleshooting
```bash
kubectl describe <resource> <name>
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top pods
kubectl top nodes
```

## Common Patterns

<!-- Add common YAML patterns for Deployments, Services, Ingress here -->
