# 🧾 Week 5 Summary Sheet — Persistence & Storage (CKAD, v1.30+)

> Scope: **PVs, PVCs, StorageClasses, AccessModes, WaitForFirstConsumer, Reclaim Policies, Labels/Selectors**
> Mode: **Exam mode** assumptions (kubectl + Vim only). Commands are exam-speed, YAML is minimal & valid.

---

## 0) Exam-day quick setup (2-minute ritual)

```bash
alias k=kubectl
source <(kubectl completion bash) 2>/dev/null || true
complete -F __start_kubectl k
k config set-context --current --namespace=wk5
k get nodes -o wide
# Vim: :set nu ts=2 sw=2 et
```

---

## 1) AccessModes vs `readOnly` mount (know this cold)

**AccessModes (PV/PVC capability & scheduling):**

* **RWO (ReadWriteOnce):** attach to **one node** at a time (many Pods on that one node can mount).
* **ROX (ReadOnlyMany):** **many nodes**, **read-only**.
* **RWX (ReadWriteMany):** **many nodes**, **read-write**.
* **RWOP (ReadWriteOncePod):** like RWO but **only one Pod** may mount at a time (strict).

**Mount `readOnly` flags (Pod/container behavior):**

* Pod-level:
  `spec.volumes[].persistentVolumeClaim.readOnly: true` → **all** containers see RO.
* Per-container:
  `spec.containers[].volumeMounts[].readOnly: true` → **that container** sees RO (others can be RW).

**Rules**

* AccessModes are **enforced by the driver/scheduler**; `readOnly: false` **cannot override** a ROX volume.
* You **can** mount an RWO claim as RO (policy allows ≥ RO).
* AccessModes are **not editable** on a bound PVC → recreate if needed.

**Tiny demo (per-container RO on RWO)**

```yaml
# One container writes, the other reads only
volumeMounts:
- { name: data, mountPath: /data, readOnly: false }   # writer
- { name: data, mountPath: /ro,   readOnly: true  }   # reader
```

---

## 2) Dynamic vs Static provisioning

**Dynamic (StorageClass set):**

* PVC specifies `storageClassName`. External provisioner **creates a new PV** that fits.
* With `volumeBindingMode: WaitForFirstConsumer`, PVC stays **Pending** until a **Pod** uses it (so the scheduler can pick a node first).
* **Reclaim policy** of the new PV comes **from the StorageClass**.

**Static (no StorageClass):**

* You create PVs **manually** (e.g., hostPath/NFS) and bind a PVC to an **existing** PV.
* Use `storageClassName: ""` on the PVC to **avoid** dynamic provisioning.

**Fast checks**

```bash
k get sc -o wide                               # see provisioner + (default) + binding mode
k describe pvc <name> | sed -n '/Events/,$p'   # will say Provisioning / WaitForFirstConsumer / why Pending
k get pv -o wide
```

---

## 3) Why **PVC selector** is ignored with StorageClass (plain English)

* **Selectors (`matchLabels`/`matchExpressions`)** on a PVC are useful **only** when you want to bind to an **existing** PV that has those labels (static path).
* If you set a **StorageClass**, Kubernetes asks the provisioner to **make a brand-new PV**. There’s **no existing PV** to match, and the provisioner **doesn’t copy your selector** into labels on the new PV.
* Result: your selector **does nothing** for dynamic provisioning. It won’t filter or constrain the new PV.

**When to use selector**

* **Static bind**: label the PV, then in the PVC:

  ```yaml
  storageClassName: ""
  selector:
    matchLabels:
      tier: gold
  ```
* **Dynamic bind**: **omit** `selector`; just set `storageClassName` (and size/mode).

---

## 4) Binding rules & common failure causes

A PVC will bind when **all** are satisfied:

* **Size**: `pv.capacity.storage` **≥** `pvc.resources.requests.storage`
* **AccessModes**: **PVC ⊆ PV** (e.g., PVC RWO binds to PV RWO+RWX)
* **volumeMode**: `Filesystem` ↔ `Filesystem` or `Block` ↔ `Block`
* **StorageClass**: equal (empty equals empty)
* **Selector** (if used): PV labels match

**Typical Pending/Fail messages to recognize**

* *“supports ReadWriteOnce and ReadWriteOncePod only”* ⇒ your SC/driver doesn’t support RWX/ROX.
* *WaitForFirstConsumer* ⇒ create the Pod; don’t over-debug the PVC.
* *no persistent volumes available for this claim* ⇒ mismatch (size, mode, class, labels).

---

## 5) Reclaim policy (what gets cleaned up when you delete the PVC?)

* **Delete** (dynamic default for many SCs): provisioner deletes the **PV object** and **backend data**.
* **Retain**: **PV object remains** in **Released** and **data stays**; manual cleanup/reuse required.
* (Recycle — deprecated; ignore for CKAD)

**Key nuance**

* A **static** hostPath PV with `Delete` **does not** delete data automatically (no provisioner owns it).

---

## 6) Speed snippets (exam-style)

**Minimal PVC (dynamic, default SC)**

> Note: Newer kubectl lacks `kubectl create pvc` generator. Write YAML & apply.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
  namespace: wk5
spec:
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 1Gi } }
  # storageClassName: <your-class>   # omit to use default
```

**Pod using a PVC**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: wk5
spec:
  volumes:
  - name: www
    persistentVolumeClaim:
      claimName: data
  containers:
  - name: nginx
    image: nginx:1.25
    volumeMounts:
    - name: www
      mountPath: /usr/share/nginx/html
```

**Static PV + PVC (label-select)**

```yaml
# PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-gold
  labels: { tier: gold }
spec:
  capacity: { storage: 1Gi }
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath: { path: /opt/k8s/pv-gold }
---
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim-gold
  namespace: wk5
spec:
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 1Gi } }
  storageClassName: ""
  selector:
    matchLabels: { tier: gold }
```

**RO mount vs AccessMode** (prove the difference)

```yaml
volumeMounts:
- name: data
  mountPath: /ro
  readOnly: true
```

```bash
k exec -n wk5 web -- sh -c 'echo hi >/ro/x'   # should fail: read-only file system
```

**Quick inspectors**

```bash
k get sc -o wide
k get pvc -n wk5 -o custom-columns=NAME:.metadata.name,SC:.spec.storageClassName,MODE:.status.accessModes,PHASE:.status.phase
k describe pvc <name> -n wk5 | sed -n '/Events/,$p'
k get pv | grep <name-or-claim>
```

---

## 7) Labels, selectors & expressions — pocket guide

**Add labels on PVs to target static binding**

```yaml
metadata:
  labels:
    pv: retain-demo
    tier: gold
```

**PVC selectors**

```yaml
selector:
  matchLabels:
    tier: gold
  matchExpressions:
  - key: pv
    operator: In          # NotIn | Exists | DoesNotExist also valid
    values: ["retain-demo"]
```

**Remember**

* **Use selectors only with static PVs** (`storageClassName: ""`).
* With **dynamic SC**, selectors are **ignored**; the brand-new PV’s labels are decided by the provisioner.

---

## 8) Gotchas you can score on

* Namespace trap: always put `metadata.namespace` on **PVCs and Pods**.
* **volumeMode mismatch**: Filesystem vs Block will keep claims Pending.
* **RWX on local-path**: will **never** bind; read Events to cite the reason.
* **Retain PV reuse**: expect **Released**; manual admin steps required to recycle.
* **WaitForFirstConsumer**: PVC Pending is normal → create the Pod.

---

## 9) What you accomplished (Week 5)

* Created **PVCs** (RWO/RWX), saw **Pending** vs **Bound** accurately.
* Mounted PVCs into Pods; **data persisted** across Pod restarts.
* Built a **static PV (Retain)** and proved data survives **PVC deletion**.
* Created a **Retain StorageClass** and verified reclaim behavior on **dynamic PVs**.
* Practiced **labels + selectors** and learned **why selectors don’t apply** with dynamic SCs.

---

## 10) One-page troubleshooting flow

1. `k describe pvc <name>` → read **Events**
2. Check SC: `k get sc -o wide`
3. If dynamic + Pending = **create Pod** (WaitForFirstConsumer)
4. If static: verify **size, AccessModes, volumeMode, labels, class ("")**
5. If RWX/ROX on local-path: **unsupported** (choose another SC)
6. Reclaim surprises? Look at PV **ReclaimPolicy** and whether it’s **dynamic vs static**
