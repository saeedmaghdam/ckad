# 📚 Week 4 Summary Sheet — Jobs & CronJobs (CKAD v1.30+)

## Exam-day 2-minute ritual

```bash
alias k=kubectl
k config get-contexts
k get nodes -o wide
# set namespace for the task set (adjust per exam question)
k create ns wk4 2>/dev/null || true
k config set-context --current --namespace wk4
```

* Use `kubectl explain <kind> --recursive | less` for fields quickly.
* Prefer **YAML via dry-run** and edit: `k create job x --image=busybox --dry-run=client -o yaml > x.yaml`.

---

## Core verification patterns

```bash
# Watch a Job’s Pods live (🔑 keep this!)
k -n wk4 get pods -l job-name=<job> -w

# Describe + logs
k -n wk4 describe job <job>
k -n wk4 logs job/<job>                 # or: k -n wk4 logs -l job-name=<job> --max-log-requests=10
```

---

## Job templates (one-off)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
  namespace: wk4
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: c
        image: busybox:1.36
        command: ["sh","-c","echo hello-ckad"]
```

**Time-box tip:** Prefer short `["sh","-c","…"]` one-liners in the exam. Multiline blocks are readable but can introduce quoting/escaping gotchas under pressure.

---

## Command vs Args (ENTRYPOINT vs CMD)

* `command:` overrides **ENTRYPOINT**; `args:` overrides **CMD**.
* Busybox often works with **args only** because its ENTRYPOINT is the multicall binary, but when unsure, use:

```yaml
command: ["sh","-c","echo hi"]
```

---

## Parallel & Indexed Jobs

**Parallel (completions vs parallelism):**

```yaml
spec:
  completions: 6
  parallelism: 3
```

**Indexed (unique shard per Pod):**

```yaml
spec:
  completionMode: Indexed
  completions: 5
  template:
    spec:
      containers:
      - name: w
        image: busybox:1.36
        command: ["sh","-c","echo I am index: $JOB_COMPLETION_INDEX"]
      restartPolicy: Never
```

---

## Retries & deadlines

**Retries:**

```yaml
spec:
  backoffLimit: 2   # total attempts = backoffLimit + 1 (here: 3)
```

**Active deadline (kill whole Job after wall time):**

```yaml
spec:
  activeDeadlineSeconds: 10
```

---

## Labels & selectors (speed)

Add your own labels to Pod template:

```yaml
template:
  metadata:
    labels: { app: batch, task: demo }
```

Then:

```bash
k -n wk4 get pods -l job-name=<job>     # controller’s auto-label (fastest)
k -n wk4 get pods -l app=batch,task=demo
```

---

## CronJob basics

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ts-cron
  namespace: wk4
spec:
  schedule: "*/2 * * * *"                 # always quote the cron string
  concurrencyPolicy: Allow
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: ts
            image: busybox:1.36
            command: ["sh","-c","echo run at: $(date -Iseconds)"]
```

**Quick checks:**

```bash
k -n wk4 get cj ts-cron -o wide
k -n wk4 get jobs -l job-name=ts-cron
k -n wk4 logs -l job-name=ts-cron --tail=10 --max-log-requests=10
```

---

## CronJob concurrency policies

* **Allow** (default): new run starts even if previous is still active.
* **Forbid**: skip if previous still active (look for `JobAlreadyActive`).
* **Replace**: controller deletes the active Job/Pod, starts a new one.

**Forcing overlap for demos:** set `sleep` longer than the interval (e.g., `sleep 90` with `*/1 * * * *`).

---

## Suspend & resume

```yaml
spec:
  suspend: true   # no new Jobs while true
```

```bash
k -n wk4 patch cj <name> -p '{"spec":{"suspend":false}}'
```

---

## startingDeadlineSeconds (missed runs)

```yaml
spec:
  startingDeadlineSeconds: 30  # skip a run if it’s >30s late
```

**To prove skip:** temporarily `suspend: true`, wait >30s past a tick, then resume—missed run won’t be backfilled.

---

## Common pitfalls (avoid in exam)

* ❌ Forgetting `metadata.namespace` → ends up in `default`.
* ❌ Omitting `restartPolicy: Never` on Jobs.
* ❌ Unquoted cron strings.
* ❌ Multiline shell with single quotes blocking `$(...)` or `$VAR`.
* ❌ Editing a Job’s Pod template and expecting retries to adopt it—**delete & reapply** the Job.

---

## Handy one-liners

```bash
# Create YAML skeletons fast
k create job j --image=busybox --dry-run=client -o yaml > j.yaml
k create cj c --image=busybox --schedule="*/5 * * * *" --dry-run=client -o yaml > c.yaml

# Logs from all pods of a job
k -n wk4 logs -l job-name=<job> --max-log-requests=10

# Two random pod logs
for p in $(k -n wk4 get pods -l job-name=<job> -o name | head -n2); do
  echo "==$p=="; k -n wk4 logs $p; 
done
```
