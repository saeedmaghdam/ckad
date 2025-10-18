# 📘 **CKAD Week 7 Summary — Services & Networking**

---

## 🧩 Core Service Types

| Type                    | Purpose                         | YAML / Imperative Shortcut                                                                                 | Verification                                                                 |
| ----------------------- | ------------------------------- | ---------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **ClusterIP (default)** | Internal cluster access         | `k expose deploy nginx --port=80 --target-port=80 --name=nginx-svc`                                        | `k exec <pod> -- wget -qO- nginx-svc:80 \| grep Welcome`                     |
| **NodePort**            | Expose on node IP (30000–32767) | `k expose deploy nginx --type=NodePort --port=80 --target-port=80 --name=nginx-nodeport --node-port=30080` | From inside Pod: `wget -qO- nginx-nodeport` ; external `curl <nodeIP>:30080` |
| **LoadBalancer**        | External LB (fronts NodePort)   | `k expose deploy nginx --type=LoadBalancer --port=80 --target-port=80 --name=nginx-lb`                     | `EXTERNAL-IP <pending>` on bare metal (✅ expected)                           |
| **Headless Service**    | DNS → Pod IPs (no ClusterIP)    | `k expose deploy nginx --port=80 --cluster-ip=None --name=nginx-headless`                                  | `k run -it --rm --image=busybox:1.36 -- nslookup nginx-headless`             |

**Exam Tip:** ClusterIP is default — you don’t need `type: ClusterIP`. Focus on labels + ports matching.

---

## 🧭 DNS & Connectivity Tests (Exam-Speed Patterns)

```bash
# 🔹 Run ephemeral BusyBox for quick checks
k run -it --rm --image=busybox:1.36 --restart=Never -- sh
# 🔹 Fetch a web page silently and show output
wget -qO- nginx-svc:80 | grep Welcome
# 🔹 Test block/fail conditions fast
wget -qO- --timeout=2 web:80 || echo BLOCKED
# 🔹 DNS lookup shortcut
nslookup nginx-headless
```

✅ **Use these repeatedly** in Pods / NetPol tests — they save 15–20 s each in exam time.

---

## 🔒 NetworkPolicy Templates & Patterns

### 🧱 1️⃣ Default Deny Ingress (“isolation base”)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: <ns>
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

👉 Blocks all traffic until explicit allow rules exist.

---

### 🎯 2️⃣ Allow Specific Pods via Label

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-granted
  namespace: <ns>
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              access: granted
      ports:
        - protocol: TCP
          port: 80
```

**Verify:**

```bash
# Before label → BLOCKED
k -n <ns> exec bb -- wget -qO- --timeout=2 web:80 || echo BLOCKED
# After label
k -n <ns> label pod bb access=granted --overwrite
k -n <ns> exec bb -- wget -qO- web:80 | grep Welcome
```

---

### 🧭 3️⃣ Allow from Specific Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-from-client-ns
  namespace: <app-ns>
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              access: client
      ports:
        - protocol: TCP
          port: 80
```

Then:

```bash
k label ns client-ns access=client --overwrite
```

✅ Pods in `client-ns` reach `web`; others blocked.

---

## ⚙️ Troubleshooting & Speed Habits

| Check                                       | Command / Reason                                                 |
| ------------------------------------------- | ---------------------------------------------------------------- |
| Endpoints exist                             | `k -n <ns> get endpoints <svc> -o wide`                          |
| NodePort reach fails externally             | check `ufw`/Proxmox firewall → open 30000-32767/TCP              |
| Confirm DNS search paths                    | `cat /etc/resolv.conf` inside Pod                                |
| Observe NetworkPolicy effects               | `k -n <ns> describe netpol <name>` → lists selected Pods & rules |
| Namespaces used in policies must be labeled | `k get ns --show-labels`                                         |

---

## 🧠 Key Exam Reminders

* Always include `metadata.namespace` in YAMLs.
* Start with **deny-all**, then open specific paths.
* **NetworkPolicies are additive** — multiple can apply.
* **Service selectors** and **Pod labels** must match exactly.
* For NodePort/LoadBalancer tasks, **functional verification inside cluster is sufficient**.
* Read **Events** in `describe` when Pods/Services misbehave — fastest debug signal.

---

## 🏁 Outcome

After Week 7 you can:

* Configure all Service types (ClusterIP → Headless).
* Verify connectivity and DNS internally.
* Implement isolation via NetworkPolicies (podSelector + namespaceSelector).
* Use rapid exam-speed commands for validation.

Your mastery level = **Intermediate+ (Exam-ready for Services & Networking)**.
