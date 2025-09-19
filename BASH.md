# 📝 CKAD CLI Training Summary

### 1. **Redirection & Pipes**

* `>` overwrite file
* `>>` append to file
* `<` read input from file
* `|` pipe output into next command
* `| tee file` show + save output

---

### 2. **Viewing & Filtering Text**

* `cat`, `less`, `head`, `tail -f` → read files/logs
* `grep "pattern" file` → search text

  * `-i` case-insensitive
  * `-n` show line numbers
  * `-v` invert match
* `cut -d' ' -f1` → extract fields by delimiter
* `awk '{print $2}'` → print specific fields

  * With condition: `awk '$2=="Running"{print $1}'`
* `jq '.items[].metadata.name'` → extract JSON fields

---

### 3. **Editing with `sed`**

* Replace first match:

  ```bash
  sed 's/old/new/' file
  ```
* Replace in-place:

  ```bash
  sed -i 's/foo/bar/' file.yaml
  ```
* Delete lines:

  ```bash
  sed '/pattern/d' file
  ```

---

### 4. **Loops & One-Liners**

* `for` loop over pods:

  ```bash
  for pod in $(kubectl get pods -o name); do
    kubectl get $pod -o wide
  done
  ```
* `while` loop for watching:

  ```bash
  while true; do kubectl get pods; sleep 2; done
  ```
* Command substitution: `$( … )`

---

### 5. **Archiving & Compression**

* Create:

  ```bash
  tar -czf archive.tar.gz mydir/
  ```
* Extract:

  ```bash
  tar -xzf archive.tar.gz
  ```
* List contents:

  ```bash
  tar -tzf archive.tar.gz
  ```

---

✅ With these tools, you can:

* Search/filter pod logs (`grep`, `awk`, `jq`)
* Make quick YAML edits (`sed`)
* Loop over many resources (`for`, `while`)
* Save and share results (pipes + redirection)
* Handle provided/extracted files (`tar`, `gzip`)
