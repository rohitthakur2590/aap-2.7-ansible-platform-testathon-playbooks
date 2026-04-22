# aap-2.7-ansible-platform-testathon-playbooks

Testathon playbooks for the **`ansible.platform` 2.7 collection** (ANSTRAT-1640).

Each playbook maps to a test case ID in the testathon tracker.  
Playbooks are self-contained: they create, assert, and clean up after themselves.  
All resources are namespaced with your `rh_username` so multiple testers can share the same AAP pod without collisions.

---

## Repository layout

```
.
├── ansible.cfg                          # inventory + stdout_callback = yaml
├── inventory                            # Three groups: aap_local, aap_http_direct, aap_http_persistent
├── group_vars/
│   ├── all.yml                          # Shared AAP credentials + rh_username (edit this)
│   ├── aap_local.yml                    # ansible_connection: local
│   ├── aap_http_direct.yml              # ansible.platform.http, persistent=false
│   └── aap_http_persistent.yml         # ansible.platform.http, persistent=true
├── vars/
│   ├── aap.yml                          # Module-param aliases (keeps vars_files working; keep in sync with group_vars/all.yml)
│   ├── user.yml                         # rh_username alias
│   └── connection.yml                   # Legacy connection mode var (unused by current playbooks)
├── reset/                               # Reusable teardown task files (included by playbooks)
├── clean_all.yml                        # Nuclear option: wipe all test resources for your username
└── playbooks/
    ├── connection/                      # Section A — Connection mode tests (target specific inventory groups)
    ├── org/                             # Section B — Organization CRUD
    ├── team/                            # Section B — Team CRUD
    ├── user/                            # Section B — User CRUD
    ├── role_definition/                 # Section B — Role Definition CRUD
    ├── role_user_assignment/            # Section B — Role User Assignment
    ├── role_team_assignment/            # Section B — Role Team Assignment
    ├── token/                           # Section B — Token lifecycle
    ├── application/                     # Section B — Application lifecycle
    ├── authenticator/                   # Section B — Authenticator lifecycle
    ├── settings/                        # Section B — Settings update/restore
    └── backward_compat/                 # Section C — 2.6 backward compatibility
```

### Inventory groups and connection modes

The inventory defines three hosts — one per connection mode. Connection parameters flow in automatically via `group_vars/` rather than being set as play variables.

| Inventory group | Host alias | `ansible_connection` | Persistent? | Used by |
|---|---|---|---|---|
| `aap_local` | `aap-local` | `local` | N/A | All Section B/C playbooks |
| `aap_http_direct` | `aap-direct` | `ansible.platform.http` | false | TC-1640-1-2 |
| `aap_http_persistent` | `aap-persistent` | `ansible.platform.http` | true | TC-1640-1-3 |

---

## Prerequisites

### 1. Install the collection

The playbooks target `ansible.platform` at the `stable-2.7` branch (or a dev build from that branch).

**From a local checkout:**

```bash
# From your ansible.platform repo root
pip install -e . --break-system-packages   # installs httpapi deps
ansible-galaxy collection build
ansible-galaxy collection install ansible-platform-*.tar.gz --force
```

**Or install directly from the branch (once published):**

```bash
ansible-galaxy collection install ansible.platform:==2.7.0 --force
```

Verify:

```bash
ansible-galaxy collection list ansible.platform
```

### 2. Configure vars

Edit **`group_vars/all.yml`** — this is the single place for all credentials and your username:

```yaml
aap_hostname: "https://my-aap-pod.example.com/"
aap_username: "admin"
aap_password: "your-password"
aap_validate_certs: false

rh_username: "jsmith"
```

> **Important:** each tester on a shared pod must set a unique `rh_username`.  
> All resources created by the playbooks include this value in their names  
> (e.g. `TestOrg-1640-jsmith`) so they cannot conflict.

The `group_vars/aap_http_direct.yml` and `group_vars/aap_http_persistent.yml` files automatically derive `ansible_host`, `ansible_user`, `ansible_password`, and `ansible_httpapi_*` from the `aap_*` vars above — you do not need to edit those files.

---

## Running playbooks locally (CLI)

```bash
# Single test case
ansible-playbook playbooks/org/TC-1640-2-1-1-create.yml

# Whole section
ansible-playbook playbooks/org/TC-1640-2-1-*.yml

# Backward compat block
ansible-playbook playbooks/backward_compat/TC-1640-3-*.yml

# Clean up everything (if a test left residue)
ansible-playbook clean_all.yml
```

### Connection mode variants

All Section B and C playbooks target `aap_local` (local connection, the default).  
The Section A playbooks target their specific inventory group — connection vars are injected automatically from `group_vars/`:

```bash
# local mode (default — all module params passed as task args)
ansible-playbook playbooks/connection/TC-1640-1-1-local.yml

# http-direct (ansible.platform.http, new session per task)
ansible-playbook playbooks/connection/TC-1640-1-2-http-direct.yml

# http-persistent (ansible.platform.http, session reused across tasks)
ansible-playbook playbooks/connection/TC-1640-1-3-http-persistent.yml
```

No extra vars or `ansible_connection` overrides needed — the inventory groups handle it.

---

## Running via AAP GUI (Job Templates)

### Step 1 — Register an Execution Environment

In the AAP GUI navigate to **Administration → Execution Environments → Add**.

| Field | Value |
|---|---|
| Name | `EE-platform-2.7-testathon` |
| Image | `<your-ee-image>` (must include `ansible.platform 2.7`) |
| Pull policy | `Always` |

### Step 2 — Create a Project

Navigate to **Resources → Projects → Add**.

| Field | Value |
|---|---|
| Name | `platform-2.7-testathon` |
| Source Control Type | `Git` |
| Source Control URL | URL of this repo |
| Branch/Tag/Commit | your branch (e.g. `aap27-testathon_playbooks`) |
| Execution Environment | `EE-platform-2.7-testathon` |

Click **Save** and wait for the project sync to complete.

### Step 3 — Create Job Templates

For each test case you want to run from the GUI, create a **Job Template**:

Navigate to **Resources → Templates → Add → Job Template**.

| Field | Value |
|---|---|
| Name | `TC-1640-2-1-1 Org Create` (match TC ID) |
| Job Type | `Run` |
| Inventory | `Demo Inventory` (or any inventory with `localhost`) |
| Project | `platform-2.7-testathon` |
| Playbook | `playbooks/org/TC-1640-2-1-1-create.yml` |
| Execution Environment | `EE-platform-2.7-testathon` |
| Variables (Extra vars) | see below |

Paste your connection details in **Extra variables**:

```yaml
aap_hostname: "https://my-aap-pod.example.com/"
aap_username: "admin"
aap_password: "your-password"
aap_validate_certs: false
rh_username: "jsmith"
```

> Alternatively, attach a **Credential** of type "AAP" and a **Survey** for `rh_username` to avoid hard-coding secrets.

---

## Test case index

### Section A — Connection Modes

| TC ID | Playbook | Description |
|---|---|---|
| TC-1640-1-1 | `playbooks/connection/TC-1640-1-1-local.yml` | `connection: local` — default httpapi |
| TC-1640-1-2 | `playbooks/connection/TC-1640-1-2-http-direct.yml` | `connection: ansible.platform.http_api` direct mode |
| TC-1640-1-3 | `playbooks/connection/TC-1640-1-3-http-persistent.yml` | `connection: ansible.platform.http_api` persistent mode |

### Section B — CRUD (core modules)

| TC ID | Playbook | Description |
|---|---|---|
| TC-1640-2-1-1 | `playbooks/org/TC-1640-2-1-1-create.yml` | Create organization + assert 2.7 & 2.6 keys |
| TC-1640-2-1-2 | `playbooks/org/TC-1640-2-1-2-idempotency.yml` | Org idempotent re-run (`changed=false`) |
| TC-1640-2-1-3 | `playbooks/org/TC-1640-2-1-3-update.yml` | Update org description + idempotency after update |
| TC-1640-2-1-4 | `playbooks/org/TC-1640-2-1-4-delete.yml` | Delete org + idempotent absent |
| TC-1640-2-2-1 | `playbooks/team/TC-1640-2-2-1-create.yml` | Create team within org |
| TC-1640-2-2-2 | `playbooks/team/TC-1640-2-2-2-update-delete.yml` | Update team description + delete |
| TC-1640-2-3-1 | `playbooks/user/TC-1640-2-3-1-create.yml` | Create user + assert output keys |
| TC-1640-2-3-2 | `playbooks/user/TC-1640-2-3-2-update-secrets.yml` | Update user email + password change |
| TC-1640-2-3-3 | `playbooks/user/TC-1640-2-3-3-delete.yml` | Delete user + idempotent absent |
| TC-1640-2-4-1 | `playbooks/role_definition/TC-1640-2-4-1-create.yml` | Create role definition with permissions |
| TC-1640-2-4-2 | `playbooks/role_definition/TC-1640-2-4-2-update-delete.yml` | Update role + delete |
| TC-1640-2-5-1 | `playbooks/role_user_assignment/TC-1640-2-5-1-assign.yml` | Assign role to user + revoke |
| TC-1640-2-6-1 | `playbooks/role_team_assignment/TC-1640-2-6-1-assign.yml` | Assign role to team + revoke |
| TC-1640-2-7-1 | `playbooks/token/TC-1640-2-7-1-create-delete.yml` | Create personal access token + delete by ID |
| TC-1640-2-8-1 | `playbooks/application/TC-1640-2-8-1-lifecycle.yml` | OAuth2 application create / update / delete |
| TC-1640-2-9-1 | `playbooks/authenticator/TC-1640-2-9-1-lifecycle.yml` | Authenticator create / enable / disable / delete |
| TC-1640-2-21-1 | `playbooks/settings/TC-1640-2-21-1-update-restore.yml` | Settings update + restore original values |

### Section C — Backward Compatibility (stable-2.6 → stable-2.7)

| TC ID | Playbook | Description |
|---|---|---|
| TC-1640-3-1 | `playbooks/backward_compat/TC-1640-3-1-flat-keys.yml` | 2.6 flat keys present on user + org output |
| TC-1640-3-2 | `playbooks/backward_compat/TC-1640-3-1-flat-keys.yml` | 2.6 flat keys present on org output (same playbook as 3-1) |
| TC-1640-3-3 | `playbooks/backward_compat/TC-1640-3-3-output-diff.yml` | All 2.6 keys present alongside new 2.7 nested keys (org, user, team) |
| TC-1640-3-4 | `playbooks/backward_compat/TC-1640-3-1-flat-keys.yml` | Round-trip: feed `result.user` back as module input; idempotent |
| TC-1640-3-5 | `playbooks/backward_compat/TC-1640-3-5-deprecated-params.yml` | Deprecated `object_id` param: task succeeds + deprecation warning issued |
| TC-1640-3-6 | `playbooks/backward_compat/TC-1640-3-6-26-playbooks-on-27.yml` | Unchanged 2.6-style playbooks run without errors on 2.7 |

---

## Tips

**Cleaning up after a failed run:**

```bash
ansible-playbook clean_all.yml
```

This removes all resources whose names contain your `rh_username`.

**Running a subset with tags (future):**  
The playbooks don't use tags yet — run them by path or glob.

**Verbose output:**

```bash
ansible-playbook playbooks/org/TC-1640-2-1-1-create.yml -v
```

**Check mode (dry run) — where supported:**

```bash
ansible-playbook playbooks/org/TC-1640-2-1-1-create.yml --check
```

Note: assertions that rely on `result.id` will be skipped in check mode since no real resource is created.

---

## Reporting results

For each TC ID, record one of:

- **PASS** — playbook completed with 0 failures, `[TC-XXXX] PASS` debug message printed
- **FAIL** — playbook hit the `rescue` block, `[TC-XXXX] FAIL` task ran; attach the full `-v` output
- **SKIP** — feature not available in this build; note the reason

File results in the shared testathon tracker (ANSTRAT-1640).
