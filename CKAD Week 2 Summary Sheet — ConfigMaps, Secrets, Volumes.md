# 📑 CKAD Week 2 Summary Sheet — ConfigMaps, Secrets, Volumes

## 🔹 ConfigMaps

**Create**

```bash
# From literals
k create cm app-config --from-literal=USER=dev --from-literal=ROLE=frontend
# From file
k create cm cfg --from-file=config.properties
```

**Use as env vars**

```yaml
env:
- name: USER
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: USER
```

**All keys as env vars**

```yaml
envFrom:
- configMapRef:
    name: app-config
```

**With prefix**

```yaml
envFrom:
- configMapRef:
    name: app-config
  prefix: APP_
```

**Mount as volume**

```yaml
volumes:
- name: cm-vol
  configMap:
    name: app-config
    items:
    - key: ROLE
      path: role.txt
      mode: 0400
containers:
- volumeMounts:
  - name: cm-vol
    mountPath: /etc/config
```

---

## 🔹 Secrets

**Create**

```bash
k create secret generic db-secret --from-literal=password=pass123
```

**As env var**

```yaml
env:
- name: DB_PASS
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

**As volume**

```yaml
volumes:
- name: sec-vol
  secret:
    secretName: db-secret
containers:
- volumeMounts:
  - name: sec-vol
    mountPath: /etc/creds
```

**⚡ Rotation behavior**

* Mounted as **volume** → auto-updates (≈60s delay).
* Injected as **env var** → requires Pod restart/rollout.

---

## 🔹 Volumes

**emptyDir (disk-backed)**

```yaml
volumes:
- name: cache
  emptyDir: {}
```

**emptyDir (RAM-backed tmpfs)**

```yaml
volumes:
- name: cache
  emptyDir:
    medium: Memory
    sizeLimit: 256Mi   # optional
```

**Shared between containers in same Pod**

```yaml
volumeMounts:
- name: cache
  mountPath: /scratch
```

---

## 🔹 Verification

* List env vars:

  ```bash
  k exec pod -- printenv | grep KEY
  ```
* Read mounted file:

  ```bash
  k exec pod -- cat /etc/config/role.txt
  ```
* Confirm mount:

  ```bash
  k exec pod -- mount | grep /cache
  ```

---

## 🔹 Exam Tips (Week 2)

* Use **imperative for simple create**, then `--dry-run=client -o yaml` → Vim edit for complex.
* **envFrom vs valueFrom**:

  * `envFrom` = all keys, fast.
  * `valueFrom` = specific key.
* For “memory-backed” volumes → `emptyDir.medium: Memory`.
* Secret rotation trap: env vars don’t update automatically.
* Always specify namespace (`-n`) or switch context early.
