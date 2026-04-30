# conserver Service Accounts

## 1. Server Account (`conserver`)

| Attribute    | Value                        |
|--------------|------------------------------|
| User         | `conserver`                  |
| Group        | `conserver`                  |
| Home         | `/nonexistent`               |
| Shell        | `/sbin/nologin`              |
| Scope        | Server only                  |
| Purpose      | Run the conserver daemon process under a restricted identity |

### 1.1. Manual Setup

```bash
groupadd conserver
useradd -r -M -d /nonexistent -g conserver -s /sbin/nologin \
        -c "conserver daemon account" conserver
```

### 1.2. Installation Path Ownership

The conserver binaries are installed under `/opt/conserver`. The daemon
process runs as `conserver:conserver` and requires read and execute
permissions on this path.

```bash
sudo chown -R root:conserver /opt/conserver
sudo chmod -R 755 /opt/conserver
```

### 1.3. systemd Service Identity

```ini
[Service]
User=conserver
Group=conserver
```

---

## 2. Node Account (`conserver-svc`)

| Attribute    | Value                        |
|--------------|------------------------------|
| User         | `conserver-svc`              |
| Group        | `ioc` (primary)              |
| Home         | `/nonexistent`               |
| Shell        | `/sbin/nologin`              |
| Scope        | Every node (IOC server)      |
| Purpose      | SSH exec target for conserver daemon to invoke `con` |

The conserver daemon on the server connects to nodes via SSH and passes
the IOC name as the SSH command. The `conserver-exec` wrapper script on
each node validates the name and executes `con -c <uds-path>`. The
`conserver-svc` account serves as the SSH landing identity. Membership
in the `ioc` group grants access to the UDS socket files (mode 0660,
owned by `ioc-srv:ioc`).

### 2.1. Manual Setup (Each Node)

```bash
useradd -r -M -d /nonexistent -g ioc -s /sbin/nologin \
        -c "conserver SSH exec target" conserver-svc
```

### 2.2. SSH Key Deployment

The server holds the private key. Each node receives the
public key in the `conserver-svc` authorized_keys file with execution
restricted to the wrapper script.

**Server (key generation, one-time):**

```bash
sudo mkdir -p /etc/conserver
sudo ssh-keygen -t ed25519 -N "" -f /etc/conserver/id_ed25519 \
     -C "conserver@central"
sudo chown conserver:conserver /etc/conserver/id_ed25519
sudo chmod 0600 /etc/conserver/id_ed25519
```

**Each node (wrapper script installation):**

A dedicated wrapper script validates the IOC name before executing `con`.
This prevents shell injection through `SSH_ORIGINAL_COMMAND`.

```bash
sudo install -m 0755 conserver-exec /usr/local/bin/conserver-exec
```

The wrapper script (`conserver-exec`) performs:
1. Rejects any input containing spaces, pipes, semicolons, or shell metacharacters
2. Validates the IOC name against `^[a-zA-Z0-9_-]+$`
3. Confirms the UDS socket file exists
4. Executes `con -c /run/procserv/<ioc-name>/control`

**Each node (SSH key installation):**

```bash
sudo mkdir -p /etc/ssh/authorized_keys.d
sudo install -m 0644 /dev/null /etc/ssh/authorized_keys.d/conserver-svc

# Paste the public key with forced-command pointing to the wrapper:
cat <<'EOF' | sudo tee /etc/ssh/authorized_keys.d/conserver-svc
restrict,command="/usr/local/bin/conserver-exec" ssh-ed25519 AAAA... conserver@central
EOF
```

The `restrict` keyword disables port forwarding, agent forwarding, PTY
allocation, and all other SSH features. The wrapper script is the sole
execution entry point.

### 2.3. sshd Configuration

If using `AuthorizedKeysFile` with a non-default path:

```
# /etc/ssh/sshd_config (or sshd_config.d/*.conf)
Match User conserver-svc
    AuthorizedKeysFile /etc/ssh/authorized_keys.d/%u
```

### 2.4. Verification

From the server, the conserver daemon sends only the IOC name
as the SSH command. Test manually:

```bash
sudo -u conserver ssh -i /etc/conserver/id_ed25519 \
     -o StrictHostKeyChecking=yes \
     -o BatchMode=yes \
     conserver-svc@<node> <ioc-name>
```

---

## 3. conserver.cf Authoring and Reload Workflow

### 3.1. Inventory Authoring

The conserver inventory is maintained as version-controlled files in a
git repository. There is no scraping of `/etc/procServ.d/` from nodes.
The repository carries:

- `site-template/etc/conserver.cf.in` -- main file (template, with `@INSTALL_LOCATION@`)
- `site-template/etc/conserver.d/<vm-name>.cf` -- one file per node, listing
  that node's IOC console blocks

Adding/removing a node = adding/removing one file under `conserver.d/` plus
the corresponding `#include` line in `conserver.cf.in`. Adding/removing IOCs
on a node = editing that node's per-node file. Every change is captured by
git history.

The site-specific production inventory lives in the site's internal git
repository. The `conserver-env` repository carries a test inventory
(`testbed-rocky8-node1.cf`, `testbed-rocky8-node2.cf`) for cloud-provision
integration testing.

### 3.2. Reload Workflow

After any cf change in the repository, the operator deploys and applies
from the server using the `cf_*` Makefile targets:

```bash
# Stages 2+3: syntax check + install to /usr/local/etc/
make cf_deploy

# Stages 4+5: syntax check installed cf + systemctl reload (SIGHUP)
make cf_apply
```

If `cf_check` fails, `cf_install` is not executed. If `cf_check_installed`
fails, `cf_reload` is not executed. Existing console sessions are
unaffected by SIGHUP -- the daemon re-reads cf without restart.

This is a manual, operator-driven process. The `ioc-runner` utility has
no dependency on or awareness of conserver.
