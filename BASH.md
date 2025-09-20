# 📝 CKAD CLI Essential Commands

## 🔎 `grep` — search text

**Syntax:**

```bash
grep [OPTIONS] "pattern" file
```

**Common flags:**

* `-i` → ignore case
* `-n` → show line numbers
* `-v` → invert match (exclude pattern)
* `-r` → recursive search in directories
* `-E` → enable extended regex (`egrep` mode)

**Example input (`pods.log`):**

```
pod-frontend Running
pod-backend CrashLoopBackOff
pod-database Running
pod-cache Running
pod-backend Error
```

**Examples:**

```bash
grep "Error" app.log             # show lines containing "Error"
grep -i "running" pods.log       # case-insensitive search
grep -v "Running" pods.log       # show lines without "Running"
grep -n "backend" pods.log       # show matches + line numbers
```

---

## ✂️ `cut` — split fields

**Syntax:**

```bash
cut -d'DELIM' -fFIELDS file
```

**Common flags:**

* `-d` → specify delimiter (default: tab)
* `-f` → choose fields

**Example input (`pods.log`):**

```
pod-frontend Running
pod-backend CrashLoopBackOff
pod-database Running
```

**Examples:**

```bash
cut -d' ' -f1 pods.log           # first column (pod name)
cut -d':' -f2 /etc/passwd        # second field from passwd file
```

---

## 📊 `awk` — field processing

**Syntax:**

```bash
awk 'condition {action}' file
```

**Common flags (rarely used, awk uses inline scripts):**

* `-F` → set delimiter (default: whitespace)

**Example input (`pods.log`):**

```
pod-frontend Running
pod-backend CrashLoopBackOff
pod-database Running
```

**Examples:**

```bash
awk '{print $2}' pods.log                       # print 2nd field
awk '$2=="Running" {print $1}' pods.log         # print 1st field if 2nd = Running
awk -F: '{print $1,$3}' /etc/passwd             # delimiter ":" → print user + UID
```

---

## 📜 `tail` — view end of file (logs!)

**Syntax:**

```bash
tail [OPTIONS] file
```

**Common flags:**

* `-n N` → show last N lines
* `-f` → follow (stream logs)
* `-F` → follow, retry if file is recreated (great for logs)

**Example input (`app.log`):**

```
[INFO] Starting server
[INFO] Listening on port 8080
[ERROR] Connection failed
[WARN] Retrying...
[INFO] Connected
```

**Examples:**

```bash
tail -n 20 app.log               # last 20 lines
tail -f app.log                  # follow live logs
tail -F /var/log/syslog          # follow even if rotated/recreated
```

---

## 🪄 `jq` — JSON query

**Syntax:**

```bash
jq 'filter' file.json
```

**Common flags:**

* `-r` → raw output (no quotes)
* `.` → identity (print whole object)

**Example input (`pods.json`):**

```json
{
  "items": [
    {"metadata": {"name": "frontend"}, "status": {"phase": "Running"}},
    {"metadata": {"name": "backend"}, "status": {"phase": "Pending"}},
    {"metadata": {"name": "database"}, "status": {"phase": "Running"}}
  ]
}
```

**Examples:**

```bash
jq '.' pods.json                             # pretty-print JSON
jq '.items[].metadata.name' pods.json        # list pod names
jq -r '.items[].metadata.name' pods.json     # list pod names (no quotes)
jq '.items[] | select(.status.phase=="Running") | .metadata.name' pods.json
```

---

# 📟 Extra: Professional Use of `screen` (Not CKAD)

`screen` lets you run multiple shell sessions inside one terminal — very useful for real-world ops work.

### 🔹 Starting & sessions

* `screen` → start a session
* `screen -S name` → start with a custom session name
* `screen -ls` → list all sessions
* `screen -r name` → reattach to session

### 🔹 Inside screen (keybindings)

Press **`Ctrl-a`** first, then:

* `c` → create a new window
* `n` → next window
* `p` → previous window
* `"` → list windows
* `A` → rename current window
* `d` → detach session (keep it running in background)

### 🔹 Killing

* `exit` → quit current screen window
* `Ctrl-a k` → kill window
* `screen -X -S name quit` → kill a detached session

### 🔹 Pro tips

* Always name sessions:

  ```bash
  screen -S ckad-labs
  ```
* Detach safely with `Ctrl-a d`, then reconnect later.
* Keep multiple screens for different tasks (logs, editing, monitoring).
