# aap-2.7-ansible-platform-testathon-playbooks

Testathon playbooks for `ansible.platform` 2.7 (ANSTRAT-1640).

Each playbook maps to a test case ID, creates resources, asserts results, and cleans up.  
All resources are namespaced with `rh_username` so multiple testers can share the same AAP pod.

---

## Getting ansible.platform 2.7

> `ansible.platform` 2.7 is **not on Galaxy yet** — do not use `ansible-galaxy collection install ansible.platform`.

### Local setup (CLI)

Clone the collection and install it:

```bash
git clone -b stable-2.7 https://github.com/ansible/ansible.platform.git
cd ansible.platform
ansible-galaxy collection build --force
ansible-galaxy collection install ansible-platform-*.tar.gz --force
```

Or install directly from the branch without cloning:

```bash
ansible-galaxy collection install \
  git+https://github.com/ansible/ansible.platform.git,stable-2.7 --force
```

Verify:
```bash
ansible-galaxy collection list ansible.platform
```

### AAP Job Templates

Use the pre-built EE `quay.io/rothakur18/platform-27-ee:latest` — has stable-2.7 pre-installed, no manual install needed.

---

## Setup

Edit **`group_vars/all.yml`** with your AAP details:

```yaml
gateway_hostname: "https://your-aap-pod.example.com/"
gateway_username: "admin"
gateway_password: "your-password"
gateway_validate_certs: false
gateway_request_timeout: 60

rh_username: "jsmith"   # unique per tester — namespaces all test resources
```

---

## Running locally

```bash
ansible-playbook playbooks/user/TC-1640-2-3-1-create.yml
ansible-playbook playbooks/org/TC-1640-2-1-*.yml
ansible-playbook clean_all.yml   # clean up after a failed run
```

---

## Running in AAP

### Inventory

Create a static inventory `testathon-inventory` with these three groups and hosts:

| Group | Host | `ansible_host` |
|---|---|---|
| `aap_local` | `aap-local` | `localhost` |
| `aap_http_direct` | `aap-direct` | `localhost` |
| `aap_http_persistent` | `aap-persistent` | `localhost` |

Set group variables (**Groups → click group → Edit variables**):

**`aap_local`:** `ansible_connection: local`

**`aap_http_direct` and `aap_http_persistent`:**
```yaml
ansible_connection: ansible.platform.http
ansible_platform_use_persistent_connection: false   # true for persistent group
ansible_host: "{{ gateway_hostname | regex_replace('^https?://', '') | regex_replace('/$', '') }}"
ansible_user: "{{ gateway_username }}"
ansible_password: "{{ gateway_password }}"
ansible_httpapi_use_ssl: true
ansible_httpapi_validate_certs: "{{ gateway_validate_certs }}"
```

### Job Template

| Field | Value |
|---|---|
| Inventory | `testathon-inventory` |
| Project | your testathon project (this repo) |
| Execution Environment | `ansible-platform-27-ee` |
| Credentials | AAP Gateway credential (injects `gateway_*` vars) |
| Extra Variables | `rh_username: yourname` |

---

## Known gotchas

**`update_secrets` and idempotency** — user module with `password:` always reports `changed: true` on re-runs because `update_secrets` defaults to `true`. Add `update_secrets: false` on idempotency check tasks.

**`del` is a reserved Python keyword** — don't use it as a `register:` variable name. Use `deleted` instead.

**"no hosts matched" in AAP** — make sure hosts are added inside each group in the inventory, not just at the top level.

**Messages like "using ephemeral manager" or "Shutting down ephemeral manager"** — informational only, not errors. Expected when running with `connection: local`.

---

## Test case index

### Section A — Connection Modes

| TC ID | Playbook |
|---|---|
| TC-1640-1-1 | `playbooks/connection/TC-1640-1-1-local.yml` |
| TC-1640-1-2 | `playbooks/connection/TC-1640-1-2-http-direct.yml` |
| TC-1640-1-3 | `playbooks/connection/TC-1640-1-3-http-persistent.yml` |

### Section B — CRUD

| TC ID | Playbook |
|---|---|
| TC-1640-2-1-1 | `playbooks/org/TC-1640-2-1-1-create.yml` |
| TC-1640-2-1-2 | `playbooks/org/TC-1640-2-1-2-idempotency.yml` |
| TC-1640-2-1-3 | `playbooks/org/TC-1640-2-1-3-update.yml` |
| TC-1640-2-1-4 | `playbooks/org/TC-1640-2-1-4-delete.yml` |
| TC-1640-2-2-1 | `playbooks/team/TC-1640-2-2-1-create.yml` |
| TC-1640-2-2-2 | `playbooks/team/TC-1640-2-2-2-update-delete.yml` |
| TC-1640-2-3-1 | `playbooks/user/TC-1640-2-3-1-create.yml` |
| TC-1640-2-3-2 | `playbooks/user/TC-1640-2-3-2-update-secrets.yml` |
| TC-1640-2-3-3 | `playbooks/user/TC-1640-2-3-3-delete.yml` |
| TC-1640-2-3-CM | `playbooks/user/TC-1640-2-3-connection-modes.yml` |
| TC-1640-2-4-1 | `playbooks/role_definition/TC-1640-2-4-1-create.yml` |
| TC-1640-2-4-2 | `playbooks/role_definition/TC-1640-2-4-2-update-delete.yml` |
| TC-1640-2-5-1 | `playbooks/role_user_assignment/TC-1640-2-5-1-assign.yml` |
| TC-1640-2-6-1 | `playbooks/role_team_assignment/TC-1640-2-6-1-assign.yml` |
| TC-1640-2-7-1 | `playbooks/token/TC-1640-2-7-1-create-delete.yml` |
| TC-1640-2-8-1 | `playbooks/application/TC-1640-2-8-1-lifecycle.yml` |
| TC-1640-2-9-1 | `playbooks/authenticator/TC-1640-2-9-1-lifecycle.yml` |
| TC-1640-2-21-1 | `playbooks/settings/TC-1640-2-21-1-update-restore.yml` |

### Section C — Backward Compatibility

| TC ID | Playbook |
|---|---|
| TC-1640-3-1 | `playbooks/backward_compat/TC-1640-3-1-flat-keys.yml` |
| TC-1640-3-3 | `playbooks/backward_compat/TC-1640-3-3-output-diff.yml` |
| TC-1640-3-5 | `playbooks/backward_compat/TC-1640-3-5-deprecated-params.yml` |
| TC-1640-3-6 | `playbooks/backward_compat/TC-1640-3-6-26-playbooks-on-27.yml` |

---

## Reporting results

- **PASS** — 0 failures, `[TC-XXXX] PASS` printed
- **FAIL** — hit rescue block; attach `-v` output
- **GAP** — works but implementation gap found; document it
- **SKIP** — not available in this build; note reason
