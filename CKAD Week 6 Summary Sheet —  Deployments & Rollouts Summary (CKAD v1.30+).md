# 📑 Week 6 — Deployments & Rollouts Summary (CKAD v1.30+)

---

## 🔧 Create Deployment

**Imperative (fast skeleton):**

```bash
k create deploy nginx-deploy --image=nginx:1.25 --replicas=3 -n wk6 \
  --dry-run=client -o yaml > d.yaml
k apply -f d.yaml
```

**Minimal YAML:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: wk6
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-deploy
  template:
    metadata:
      labels:
        app: nginx-deploy
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

---

## 📈 Scaling

```bash
k -n wk6 scale deploy nginx-deploy --replicas=5
k -n wk6 get deploy nginx-deploy
```

---

## 🔄 Image Updates & Rollouts

**Update image + watch:**

```bash
k -n wk6 set image deploy/nginx-deploy nginx=nginx:1.27
k -n wk6 rollout status deploy/nginx-deploy
```

**Rollback:**

```bash
k -n wk6 rollout undo deploy/nginx-deploy
k -n wk6 rollout undo deploy/nginx-deploy --to-revision=1
```

**History with cause:**

```bash
k -n wk6 annotate deploy/nginx-deploy kubernetes.io/change-cause="Upgrade to 1.27" --overwrite
k -n wk6 rollout history deploy/nginx-deploy
```

---

## ⏯️ Pause & Resume

```bash
k -n wk6 rollout pause deploy/nginx-deploy
# make changes while paused
k -n wk6 rollout resume deploy/nginx-deploy
k -n wk6 rollout status deploy/nginx-deploy
```

---

## ⚖️ Strategies

**Default = RollingUpdate (configurable):**

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%
    maxSurge: 25%
```

**Recreate:**

```yaml
strategy:
  type: Recreate
```

---

## 📝 Quick Verifications

**Check current image:**

```bash
k -n wk6 get deploy nginx-deploy -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

**Check ReplicaSets:**

```bash
k -n wk6 get rs -l app=nginx-deploy
```

---

## 🔄 RollingUpdate vs Recreate (Easy Explanation)

* **RollingUpdate (default)**

  * Replace Pods **gradually**, old + new run together.
  * Controlled by `maxUnavailable` + `maxSurge`.
  * Goal: **zero downtime** if app tolerates mixed versions.
  * Example: 5 Pods → update → 4 old + 1 new → 3 old + 2 new → … until all updated.

* **Recreate**

  * Kill **all old Pods first**, then start new ones.
  * Causes **downtime** during update.
  * Use if old & new cannot coexist (e.g., schema migrations).
  * Example: 5 old → 0 → 5 new.

✅ **Exam memory hook:**
*RollingUpdate = smooth & safe. Recreate = simple but downtime.*

---

# 🔑 Exam Tips Recap

* Always match **name + namespace** exactly.
* Use **imperative + dry-run** to save time.
* Prefer **JSONPath** over long `describe`.
* Annotate with `kubernetes.io/change-cause` for readable history.
* Remember: default strategy = **RollingUpdate**.
