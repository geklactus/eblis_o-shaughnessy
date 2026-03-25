# GitLab Runner Deployment — RHEL 10 on ESX

Ansible playbook to deploy a fully configured GitLab Runner (Docker executor) on a
Red Hat Enterprise Linux 10 VM running on VMware ESX, connecting to an existing
GitLab instance over SSL using InCommon certificates.

---

## What This Deploys

| Component | Details |
|---|---|
| **Static IP** | Configures NetworkManager via `nmcli` on the specified interface |
| **SSL Certificates** | Auto-detects and deploys PEM server cert, private key, and CA/intermediate chain from a source directory |
| **SELinux** | Sets enforcing mode with container-specific policies |
| **Firewall** | Configures `firewalld` with SSH, HTTPS, Docker bridge, and masquerading |
| **Docker CE** | Installs Docker CE with overlay2 storage and JSON log driver |
| **GitLab Runner** | Installs, configures, and auto-registers the runner with your GitLab instance |
| **Log Rotation** | Configures `logrotate` for both GitLab Runner and Docker container logs |

---

## Prerequisites

1. **Ansible 2.15+** on your control machine
2. **SSH access** to the target VM (key-based recommended)
3. **Ansible collections** — install before running:

   ```bash
   ansible-galaxy install -r requirements.yml
   ```

4. **InCommon certificate files** (PEM format) in a directory on the controller:
   - `server.crt` (or any `.crt` — the server certificate)
   - `server.key` (or any `.key` — the private key)
   - `incommon-intermediate.crt` / `chain.pem` (CA/intermediate certificates)

   The playbook auto-detects which file is which by inspecting the PEM content.

5. **GitLab Runner registration token** from your GitLab instance
   (Settings → CI/CD → Runners → New project runner)

---

## Quick Start

### 1. Configure Variables

Edit `group_vars/all.yml` and set every value marked `CHANGE ME`:

```yaml
# GitLab
gitlab_url:           "https://gitlab.yourorg.edu"
gitlab_runner_token:  "glrt-xxxxxxxxxxxxxxxxxxxx"

# Network
network_interface:    "ens192"
network_ip_address:   "10.0.1.50"
network_prefix:       24
network_gateway:      "10.0.1.1"
network_dns_servers:  ["10.0.1.2", "10.0.1.3"]
network_dns_search:   ["yourorg.edu"]

# Certificates (path on the Ansible controller)
cert_source_dir:      "/home/admin/certs/gitlab-runner"
```

### 2. Set the Inventory

Edit `inventory/hosts.yml`:

```yaml
gitlab_runners:
  hosts:
    gitlab-runner-01:
      ansible_host: "192.168.1.100"    # current reachable IP
      ansible_user: "ansible"
```

### 3. Run the Playbook

```bash
# Full deployment
ansible-playbook site.yml

# With the token passed securely at runtime
ansible-playbook site.yml -e "gitlab_runner_token=glrt-xxxxxxxxxxxxxxxxxxxx"

# Dry run
ansible-playbook site.yml --check --diff

# Run only specific components
ansible-playbook site.yml --tags certificates
ansible-playbook site.yml --tags "docker,gitlab_runner"
```

---

## Certificate Auto-Detection

The `certificates` role automatically inspects every `.pem`, `.crt`, `.key`, and
`.cer` file in `cert_source_dir` and classifies them:

| Classification | How It's Detected |
|---|---|
| **Private Key** | Contains `PRIVATE KEY` PEM header |
| **Server Certificate** | X.509 cert without `CA:TRUE` in basicConstraints |
| **CA / Intermediate** | X.509 cert with `CA:TRUE` in basicConstraints |

The role then:

1. Deploys the key to `/etc/pki/tls/private/server.key` (mode 0600)
2. Deploys the server cert to `/etc/pki/tls/certs/server.crt`
3. Deploys CA/intermediates to `/etc/pki/ca-trust/source/anchors/`
4. Runs `update-ca-trust extract` to rebuild the system trust store
5. Assembles a full-chain PEM (server + intermediates)
6. Copies the full chain into `/etc/gitlab-runner/certs/<gitlab-hostname>.crt`
   so GitLab Runner trusts your GitLab instance's certificate

---

## Directory Structure

```
gitlab-runner-deploy/
├── ansible.cfg
├── requirements.yml
├── site.yml                          # Main playbook
├── inventory/
│   └── hosts.yml
├── group_vars/
│   └── all.yml                       # ← All configurable variables
└── roles/
    ├── network/                      # Static IP via NetworkManager
    ├── certificates/                 # InCommon SSL auto-detect & deploy
    ├── selinux/                      # SELinux enforcing + Docker policies
    ├── firewall/                     # firewalld rules
    ├── docker/                       # Docker CE installation
    ├── gitlab_runner/                # Runner install + auto-registration
    └── logrotate/                    # Log rotation configs
```

---

## Available Tags

Run subsets of the playbook with `--tags`:

| Tag | Roles |
|---|---|
| `network`, `ip` | Static IP configuration |
| `certificates`, `ssl` | Certificate deployment |
| `selinux`, `security` | SELinux configuration |
| `firewall`, `security` | Firewall rules |
| `docker` | Docker CE |
| `gitlab_runner`, `runner` | GitLab Runner install + registration |
| `logrotate`, `logging` | Log rotation |

---

## Security Notes

- The runner token is marked `no_log: true` in the registration task
- Private keys are deployed with mode `0600` and SELinux type `cert_t`
- Docker runs unprivileged by default (`privileged = false`)
- SELinux is set to `enforcing` with a minimal custom policy module
- Consider using **Ansible Vault** for the runner token:

  ```bash
  ansible-vault encrypt_string 'glrt-xxxxxxxxxxxxxxxxxxxx' --name 'gitlab_runner_token'
  ```

---

## Post-Deployment Verification

After a successful run, verify from the target host:

```bash
# Check runner status
sudo gitlab-runner status
sudo gitlab-runner list

# Verify SSL connectivity to GitLab
openssl s_client -connect gitlab.yourorg.edu:443 -CApath /etc/pki/tls/certs/

# Check Docker
docker info
docker ps

# Check SELinux
getenforce
sestatus

# Check firewall
firewall-cmd --list-all
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Runner can't reach GitLab | Check `firewall-cmd --list-all` for HTTPS; verify DNS in `/etc/resolv.conf` |
| TLS certificate errors | Ensure the full chain is at `/etc/gitlab-runner/certs/<hostname>.crt`; run `update-ca-trust extract` |
| Docker permission denied | Verify `gitlab-runner` is in the `docker` group: `id gitlab-runner` |
| SELinux AVC denials | Check `ausearch -m avc -ts recent` and add rules to the `.te` template |
| Network role hangs | The static IP change may break the current SSH session; re-run targeting the new IP |
