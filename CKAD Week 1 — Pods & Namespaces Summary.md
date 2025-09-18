# 📑 CKAD Week 1 — Pods & Namespaces Summary

---

## 🟢 Pod Creation

### Imperative (fastest)

```bash
# Nginx pod
kubectl run nginx-pod --image=nginx:1.25

# Busybox pod with command
kubectl run busybox-pod --image=busybox:1.36 --command -- sleep 3600
```

### YAML Generation

```bash
kubectl run mypod --image=nginx:1.25 --dry-run=client -o yaml > pod.yaml
kubectl apply -f pod.yaml
```

---

## 🟢 Multi-Container Pod (YAML)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  containers:
  - name: nginx-container
    image: nginx:1.25
  - name: busybox-container
    image: busybox:1.36
    command: ["sleep", "3600"]
```

Apply:

```bash
kubectl apply -f multi-pod.yaml
kubectl describe pod multi-pod
```

---

## 🟢 Environment Variables

### Imperative

```bash
kubectl run env-pod --image=nginx:1.25 --env=MODE=dev
```

### YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    env:
    - name: MODE
      value: dev
```

Verify:

```bash
kubectl exec -it env-pod -- printenv | grep MODE
```

---

## 🟢 Namespaces

```bash
# Create namespace
kubectl create ns review-ns

# List namespaces
kubectl get ns

# Switch current context to namespace
kubectl config set-context --current --namespace=review-ns

# Run pod in namespace
kubectl run httpd-review --image=httpd:2.4 -n review-ns
kubectl get pods -n review-ns
```

---

## 🟢 Debugging Pods

```bash
# Exec into a container
kubectl exec -it multi-pod -c busybox-container -- sh

# Run command inside
echo CKAD > /tmp/ckad.txt
cat /tmp/ckad.txt
```

---

## 🟢 Verification Patterns

```bash
kubectl get pods
kubectl get pods -n <ns>
kubectl describe pod <pod>
kubectl logs <pod>
```
