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

## 3. conserver.cf Generator and Reload Workflow

### 3.1. Generator Location

The `conserver.cf` generator resides in the `conserver-env` repository
alongside the conserver build and deployment infrastructure. It parses
IOC configuration files (`/etc/procServ.d/*.conf`) collected from all
nodes to produce a complete `conserver.cf`.

### 3.2. IOC Configuration Collection

The generator requires access to every node's `/etc/procServ.d/`
directory. Two collection methods:

**Method A — SSH pull (recommended):**

```bash
for host in proton electron kaon; do
    scp -i /etc/conserver/id_ed25519 \
        conserver-svc@${host}:/etc/procServ.d/*.conf \
        /etc/conserver/inventory/${host}/
done
```

**Method B — Shared filesystem (NFS):**

If `/etc/procServ.d/` is already on a shared mount, no collection step
is needed.

### 3.3. Reload Workflow

After `ioc-runner install` or `ioc-runner remove` on any node,
an operator regenerates and applies the conserver configuration from
the server:

```bash
# 1. Collect current IOC configurations
generate-conserver-cf

# 2. Validate
/opt/conserver/sbin/conserver -t -C /etc/conserver.cf.new

# 3. Apply (no daemon restart required)
sudo mv /etc/conserver.cf.new /etc/conserver.cf
sudo kill -HUP $(pidof conserver)
```

This is a manual, operator-driven process. The `ioc-runner` utility has
no dependency on or awareness of conserver.
