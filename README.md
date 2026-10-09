# Scope-file-Generate
## Usage

### From the command line — one domain

```bash
cd ~/Desktop/New_Folder
python3 mkscope.py --program acme --domain acme.test
```

Produces `config/scope.yaml` containing:

```yaml
allowed:
  - "acme.test"
  - "*.acme.test"
```

### Multiple domains

```bash
python3 mkscope.py --program acme \
  --domain acme.test \
  --domain api.acme.test \
  --domain cdn.acme.test
```

### From a list file

Create `data/domains.txt`:

```
# acme program seed
acme.test
api.acme.test      # API host
cdn.acme.test
```

Run:

```bash
python3 mkscope.py --program acme --domains-file data/domains.txt
```

### With exclusions

```bash
python3 mkscope.py --program acme \
  --domain acme.test \
  --exclude admin.acme.test \
  --exclude billing.acme.test
```

### IPs and CIDR blocks

```bash
python3 mkscope.py --program acme \
  --domain 203.0.113.10 \
  --domain 198.51.100.0/24
```

IPs and CIDR blocks are written verbatim — no wildcard is generated for them.

### Write to a different file

```bash
python3 mkscope.py --program acme --domain acme.test \
  --output config/scope-acme.yaml
```

### Overwrite an existing file

```bash
python3 mkscope.py --program acme --domain acme.test --force
```

Without `--force`, the script refuses to overwrite.

### Suppress wildcard expansion

```bash
python3 mkscope.py --program acme --domain acme.test --no-wildcard
```

Produces only `acme.test` (apex, no subdomains).

---

## What the script guarantees

1. **`policy_reviewed: false`** — always. The script will not set it to `true` for you. That flag means you personally read the policy.
2. **`active_checks: false`** — always. Same reasoning.
3. **Conservative defaults** — 1 req/s, 2 concurrent, 200-request budget, 512 KB response cap.
4. **Refuses to overwrite** unless `--force` is passed.
5. **Fails on empty input** — will not produce a scope with zero allowed entries.
6. **Normalizes input** — strips `https://`, paths, ports, and trailing dots, and lower-cases.
7. **Validates IP/CIDR** — a malformed CIDR is rejected, not written.

---

## What it does not do

- It does not fetch the program policy. There is no public HackerOne API for scope, and even if there were, importing a scope silently would defeat the point of a review step.
- It does not guess which subdomains are sensitive. Exclusions must be supplied by you.
- It does not verify that a domain is actually in scope. That check happens when you run `h1hunter scope check` on a target.

---

## Verify the output

```bash
h1hunter scope validate --file config/scope.yaml

h1hunter scope check --file config/scope.yaml --target https://api.acme.test/
h1hunter scope check --file config/scope.yaml --target https://admin.acme.test/
h1hunter scope check --file config/scope.yaml --target https://random.test/
```

Expected:

```
ALLOWED  https://api.acme.test/     — matches *.acme.test
DENIED   https://admin.acme.test/   — matches exclusion
DENIED   https://random.test/       — no matching allowed entry
```

If those three do not match expectations, edit `scope.yaml` and re-validate.

---

## Optional: make it executable

```bash
chmod +x mkscope.py
./mkscope.py --program acme --domain acme.test
```

Or install it as a `h1hunter` companion script by adding to `pyproject.toml`:

```toml
[project.scripts]
h1hunter = "h1hunter.cli:main"
mkscope  = "scripts.mkscope:main"
```

…and moving the file to `src/scripts/mkscope.py`. That is optional; running `python3 mkscope.py` is fine for a personal tool.

---

## Summary

One command generates a valid, conservative scope file from a domain list:

```bash
python3 mkscope.py --program <name> --domains-file data/domains.txt
```

It writes `config/scope.yaml`, expands bare domains to wildcards, keeps `policy_reviewed` and `active_checks` at `false`, and refuses to overwrite without `--force`. You still have to open the file, verify each entry against the program page, and set the two booleans yourself — the tool cannot and must not do that on your behalf.
