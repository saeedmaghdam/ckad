# 📌 Week 3 Summary Sheet — Init, Probes, Multi-container Patterns

## 1) Quick Exam Ritual (2 min)

```bash
alias k=kubectl
source <(kubectl completion bash) 2>/dev/null || true
complete -F __start_kubectl k
k config get-contexts
k get nodes -o wide
# Vim: :set nu ts=2 sw=2 et
```

## 2) Init Containers

```yaml
apiVersion: v1
kind: Pod
metadata: {name: web-init}
spec:
  volumes:
  - name: workdir
    emptyDir: {}
  initContainers:
  - name: init
    image: busybox:1.36
    command: ["sh","-c","echo Init > /work/init.txt"]
    volumeMounts: [{name: workdir, mountPath: /work}]
  containers:
  - name: app
    image: nginx:1.25
    volumeMounts: [{name: workdir, mountPath: /work}]
```

Verify:

```bash
k describe pod web-init | sed -n '/Init Containers:/,/Containers:/p'
k exec web-init -- cat /work/init.txt
```

## 3) Probes (readiness, liveness, startup)

```yaml
containers:
- name: web
  image: nginx:1.25
  readinessProbe:
    httpGet: {path: "/", port: 80}
    initialDelaySeconds: 2
    periodSeconds: 3
  livenessProbe:
    httpGet: {path: "/", port: 80}
    periodSeconds: 5
  startupProbe:
    httpGet: {path: "/", port: 80}
    initialDelaySeconds: 10
    periodSeconds: 5
    failureThreshold: 10   # ~50s grace
```

Rules of thumb:

* Readiness fail ⇒ **no traffic**, no restart.
* Liveness fail ⇒ **restart**.
* Startup ⇒ disables liveness until startup passes.

## 4) Sidecar (log tailing)

```yaml
volumes: [{name: logs, emptyDir: {}}]
containers:
- name: web
  image: nginx:1.25
  volumeMounts: [{name: logs, mountPath: /var/log/nginx}]
- name: tailer
  image: busybox:1.36
  command: ["sh","-c"]
  args: ["tail -n+1 -f /var/log/nginx/access.log"]
  volumeMounts: [{name: logs, mountPath: /var/log/nginx}]
```

Verify:

```bash
POD_IP=$(k get pod <pod> -o jsonpath='{.status.podIP}')
for i in 1 2 3; do wget -qO- http://$POD_IP >/dev/null; done
k logs -f <pod> -c tailer --tail=10
```

## 5) Ambassador (local proxy)

Minimal header-correct version (no `nc -e` required):

```yaml
- name: ambassador
  image: busybox:1.36
  command: ["sh","-c"]
  args:
  - |
    while true; do
      { printf "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\nConnection: close\r\n\r\n";
        wget -qO- http://example.com; } | nc -l -p 8080 -q 1
    done
  ports: [{containerPort: 8080}]
```

App uses `http://127.0.0.1:8080`.

## 6) Adapter (transform stream)

```yaml
volumes: [{name: shared, emptyDir: {}}]
containers:
- name: producer
  image: busybox:1.36
  command: ["sh","-c"]
  args: ['id=1; while true; do echo "{\"id\":$id,\"ts\":\"$(date)\"}" >> /data/out.log; id=$((id+1)); sleep 1; done']
  volumeMounts: [{name: shared, mountPath: /data}]
- name: adapter
  image: busybox:1.36
  command: ["sh","-c"]
  args: ['tail -n+1 -f /data/out.log | awk -F\" '"'"'/"ts"/{for(i=1;i<=NF;i++)if($i=="ts"){print $(i+2); fflush(); break}}'"'"'']
  volumeMounts: [{name: shared, mountPath: /data}]
```

## 7) JSONPath cheats (super fast)

```bash
# Pods
k get po <p> -o jsonpath='{.status.podIP}{"\n"}'
k get po <p> -o jsonpath='{.status.containerStatuses[*].ready}{"\n"}'
k get po <p> -o jsonpath='{.spec.containers[*].image}{"\n"}'

# Probes exist?
k get po <p> -o jsonpath='{.spec.containers[?(@.name=="web")].readinessProbe.httpGet.path}{"\n"}'
```

## 8) Imperative starters (muscle memory)

```bash
k run NAME --image=nginx:1.25 --dry-run=client -o yaml > pod.yaml
k create deploy web --image=nginx:1.25 --dry-run=client -o yaml > deploy.yaml
k expose deploy web --port 80 --target-port 80 --dry-run=client -o yaml > svc.yaml
```
