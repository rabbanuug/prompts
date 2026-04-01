---
trigger: model_decision
description: Follow these Ansible standards for all YAML, playbooks, roles, and configs Focus on idempotency, secure secret handling (Vault), and clean variable management. Prioritize modular roles over monolithic playbooks and avoid shell/command anti-patterns.
---

glob: "**/playbooks/**/*.yml,**/roles/**/*.yml,**/tasks/**/*.yml,**/handlers/**/*.yml,**/group_vars/**,**/host_vars/**,**/inventory/**,**/collections/requirements.yml,**/requirements.yml,**/.ansible-lint,**/ansible.cfg,**/molecule/**/*.yml,site.yml"


# Ansible — Best Practices & Anti-Patterns

> **Baseline:** ansible-core 2.16+ · Collections-first model.
> `ansible.builtin.include` **removed in 2.16** — never use it.
> `sudo:` keyword **removed** — use `become:` / `become_user:`.
> `with_*` loops **deprecated** — use `loop:` with filters.

---

## Anti-Patterns ❌

### Bare `include:` / `sudo:` / `with_*`
```yaml
# WRONG
- include: tasks/setup.yml      # removed in 2.16
  sudo: yes                     # removed
  with_items: "{{ packages }}"  # deprecated
# CORRECT
- ansible.builtin.import_tasks: tasks/setup.yml     # static — parsed at load time
- ansible.builtin.include_tasks: "tasks/{{ env }}.yml"  # dynamic — runtime
  loop: "{{ packages }}"
```
`import_*` = static (no loops/conditionals on statement). `include_*` = dynamic (supports loop/when, but tags/handlers don't propagate upstream).

### FQCN missing
```yaml
# WRONG — ambiguous, breaks in EEs
- copy:
    src: foo.conf
    dest: /etc/app/foo.conf
# CORRECT
- ansible.builtin.copy:
    src: foo.conf
    dest: /etc/app/foo.conf
```
All modules must use FQCN. Triggers ansible-lint `fqcn` rule. No exceptions.

### Global `become: yes` on playbook
```yaml
# WRONG — root for every task
- hosts: webservers
  become: yes
# CORRECT — scope per task
- hosts: webservers
  tasks:
    - ansible.builtin.package:
        name: nginx
        state: present
      become: yes
    - ansible.builtin.git:
        repo: https://github.com/org/app.git
        dest: /opt/app
      become: yes
      become_user: appuser
```

### Hardcoded secrets
```yaml
# WRONG
db_password: "s3cr3t"
# CORRECT — vault or external lookup
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...
# AWS-native alternative
- ansible.builtin.set_fact:
    db_creds: "{{ lookup('amazon.aws.secretsmanager_secret',
                  'prod/db/creds', region='us-east-1') | from_json }}"
```
Never commit `.vault_pass`. Never pass secrets via `-e "key=value"` (leaks to process list). Use `no_log: true` on tasks that register sensitive values.

### `command:` / `shell:` when a module exists
```yaml
# WRONG
- ansible.builtin.shell: "systemctl restart nginx"
# CORRECT
- ansible.builtin.service:
    name: nginx
    state: restarted
  become: yes
```
`shell:` and `command:` are not idempotent. When unavoidable, add `changed_when:` and `failed_when:`.

### Variables defined in plays
```yaml
# WRONG
- hosts: webservers
  vars:
    nginx_port: 80
# CORRECT — group_vars/webservers/nginx.yml
nginx_port: 80
nginx_app_user: deploy   # always prefix with role name
```

### `host_key_checking = False`
```ini
# WRONG — disables MITM protection
[defaults]
host_key_checking = False
# CORRECT
[ssh_connection]
ssh_args = -o StrictHostKeyChecking=yes -o UserKnownHostsFile=~/.ssh/known_hosts
```

### Unversioned collections / roles
```yaml
# WRONG
- src: geerlingguy.docker
# CORRECT — collections/requirements.yml
collections:
  - name: community.docker
    version: "3.10.4"
  - name: amazon.aws
    version: "8.2.0"
```

### `state: latest` for packages
```yaml
# WRONG — breaks idempotency
- ansible.builtin.package:
    name: nginx
    state: latest
# CORRECT
    state: present   # or pin version: "nginx-1.24.*"
```

---

## Best Practices ✅

### Directory Layout
```
inventory/
  production.ini
  staging.ini
  dynamic/            # dynamic inventory plugin configs (aws_ec2.yml etc.)
group_vars/
  all/
    main.yml          # global defaults
    vault.yml         # encrypted secrets ONLY
  webservers/
    nginx.yml         # named after role
host_vars/
  web01.prod.yml
collections/
  requirements.yml    # pin all collections with version
roles/
  requirements.yml    # pin all Galaxy roles with version
  internal/
  external/           # downloaded via ansible-galaxy (gitignored)
playbooks/
  site.yml            # orchestration only — no tasks
  webservers.yml
library/              # custom modules
filter_plugins/
molecule/
ansible.cfg
.ansible-lint
```

### ansible.cfg Hardening
```ini
[defaults]
inventory                = inventory/
roles_path               = roles/internal:roles/external
remote_user              = ansible-svc
stdout_callback          = yaml
callbacks_enabled        = timer, profile_tasks
deprecation_warnings     = True
error_on_undefined_vars  = True

[privilege_escalation]
become        = False     # off by default; enable per-task
become_method = sudo

[ssh_connection]
pipelining = True         # requires requiretty=False in sudoers
ssh_args   = -o StrictHostKeyChecking=yes -o ControlMaster=auto -o ControlPersist=60s
retries    = 3
```
Never set `log_path` without securing the file (0600, ansible-owned) — sensitive data leaks there.

### Role Best Practices
- One purpose per role. Split `install`, `configure`, `service` concerns.
- All role variables prefixed with role name: `nginx_port`, not `port`.
- `defaults/main.yml` = user-overridable. `vars/main.yml` = role-internal constants.
- Use `ansible-galaxy role init` — never hand-create skeleton.
- `meta/main.yml` must declare `min_ansible_version` and `dependencies`.

### Playbook Structure
```yaml
---
- name: Configure web tier
  hosts: webservers
  gather_facts: true
  any_errors_fatal: false
  serial: "20%"           # rolling update

  pre_tasks:
    - name: Assert minimum Ansible version
      ansible.builtin.assert:
        that: "ansible_version.full is version('2.16', '>=')"
        fail_msg: "Requires ansible-core >= 2.16"

  roles:
    - role: common
      tags: [common]
    - role: nginx
      tags: [nginx, web]

  post_tasks:
    - name: Verify nginx listening
      ansible.builtin.wait_for:
        port: 80
        timeout: 30
      tags: [verify]
```

### Task Authoring
```yaml
- name: Deploy app config        # always name tasks
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/app/app.conf
    owner: appuser
    group: appuser
    mode: "0640"                 # always set mode on files
  notify: Restart app
  tags: [app, config]

# block/rescue for rollback
- name: Deploy with rollback
  block:
    - ansible.builtin.template:
        src: app.conf.j2
        dest: /etc/app/app.conf
        mode: "0640"
    - ansible.builtin.service:
        name: app
        state: restarted
      become: yes
  rescue:
    - ansible.builtin.copy:
        src: /etc/app/app.conf.bak
        dest: /etc/app/app.conf
        remote_src: yes
```

### Handlers
```yaml
# handlers/main.yml
- name: Restart nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
  become: yes
  listen: "restart web services"   # use listen for grouped triggers
```
Handlers run once at play end. Use `flush_handlers` mid-play only when ordering requires it.

### Secrets — Double-Var Pattern
```yaml
# group_vars/production/vault.yml (ansible-vault encrypted)
vault_db_password: "..."
# group_vars/production/main.yml (plain text, refs vault vars)
db_password: "{{ vault_db_password }}"
```
Use `ansible-vault encrypt_string` for individual values — easier to diff in PRs.
In CI: `ANSIBLE_VAULT_PASSWORD_FILE` env var pointing to a tempfile; wipe after run.

### Inventory Rules
- One inventory file per environment. Never mix prod and staging.
- For AWS/EKS: use `amazon.aws.aws_ec2` dynamic inventory plugin, not static INI.
- Never set `ansible_password` or `ansible_become_password` in inventory — use vault vars.
- Group by purpose, geography, and environment.

### Idempotency Checklist
- `command:` / `shell:` tasks require `creates:`, `removes:`, or `changed_when:` / `failed_when:`.
- Avoid `state: latest` — non-idempotent.
- Template changes trigger handler (reload/restart), not direct task.
- Validate: run playbook twice; second run must show 0 changes.

### Testing — Molecule
```yaml
# molecule/default/molecule.yml
driver:
  name: docker          # or podman for rootless
platforms:
  - name: ubuntu2204
    image: quay.io/ansible/ubuntu2204-test-container:latest
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible         # verify.yml asserts actual state
lint: |
  set -e
  yamllint .
  ansible-lint .
```
Every role needs a Molecule scenario. `verify.yml` must assert real state (port open, perms, service running). Use `quay.io/ansible/*-test-container` — includes systemd.

### Linting — `.ansible-lint`
```yaml
profile: production     # enforces fqcn, no-log-password, risky-shell-pipe
exclude_paths:
  - roles/external/
```
Run in pre-commit and CI. Never add `# noqa` without an explanatory comment.

### Security Checklist
- `no_log: true` on tasks registering/printing passwords, tokens, keys.
- Dedicated `ansible-svc` OS user; sudoers scoped to specific binaries only.
- Verify downloaded binaries with checksum:
  ```yaml
  - ansible.builtin.get_url:
      url: "https://example.com/tool"
      dest: /tmp/tool
      checksum: "sha256:{{ expected_sha256 }}"
  ```
- Rotate vault encryption keys on schedule; `ansible-vault rekey`.
- Use `ansible-sign` to sign collections in supply-chain-sensitive environments.

### Execution Environments (Modern Runtime)
```yaml
# execution-environment.yml
version: 3
dependencies:
  galaxy: collections/requirements.yml
  python: requirements.txt
images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-24/ee-minimal-rhel9:latest
```
Build with `ansible-builder`. Run with `ansible-navigator`. Pin base image digest. Bake all Python deps into the EE image — never install at playbook runtime.