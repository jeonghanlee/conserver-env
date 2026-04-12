# EPICS IOC conserver Integration Architecture

## 1. Overview

This document describes the architecture for integrating `conserver` with the
`epics-ioc-runner` management environment. The goal is to provide centralized,
multi-user console access to EPICS IOCs running under `procServ` across multiple
nodes, without modifying the existing `ioc-runner` deployment model.

**Terminology:**

| Term       | Description                                              |
|------------|----------------------------------------------------------|
| **server** | Central server running the conserver daemon + `console`  |
| **node**   | IOC server running procServ + ioc-runner + con           |

---

## 2. Base Environment

| Component       | Detail                                          |
|-----------------|-------------------------------------------------|
| Nodes           | Debian 13 / Rocky 8.10                          |
| Server          | Debian 13 / Rocky 8.10 (conserver daemon)       |
| IOC management  | systemd + procServ + ioc-runner                 |
| IOC console     | con (UNIX Domain Socket client)                 |
| UDS path        | `/run/procserv/<ioc-name>/control` (mode 0660)  |
| Access control  | `ioc` group + `/etc/sudoers.d/10-epics-ioc`     |
| conserver IPC  | UDS `/run/conserver` (no TCP listener)          |

---

## 3. Architecture

```
[ Engineer ]
     |
     | ssh to server, then: console <ioc-name>
     |
[ Server ] Debian 13 / Rocky 8.10
  - conserver daemon (1 instance)
  - console command (conserver client)
  - SSH key: /etc/conserver/id_ed25519
  - bin/generate-conserver-cf
     |
     | type exec + SSH (key-based, BatchMode)
     | ssh conserver-svc@<node> <ioc-name>
     |
[ Node N ] Debian 13 / Rocky 8.10
  - bin/conserver-exec    (SSH forced-command wrapper, validates IOC name)
  - con                   (local UDS client, invoked by conserver-exec)
  - procServ  -->  /run/procserv/<ioc-name>/control  (UDS, 0660, ioc group)
  - systemd:       epics-@<ioc-name>.service
  - /etc/procServ.d/<ioc-name>.conf
```

---

## 4. Key Design Decisions

**conserver uses `type exec` + SSH, not `type uds`.**
`type uds` is local-only. conserver reaches nodes over SSH and passes the
IOC name as the SSH command. This requires no new open ports on nodes and
reuses the existing SSH infrastructure.

**Client-daemon communication uses UDS, not TCP.**
conserver is built with `--with-uds=/run/conserver` and
`--with-trust-uds-cred`. The `console` client connects to the daemon
via `/run/conserver` instead of TCP 7782. Since both the client and daemon
run on the same server, TCP exposure is unnecessary. UDS eliminates
network-level attack surface and enables `SO_PEERCRED` identity verification.

**SSH forced-command delegates to a wrapper script (`conserver-exec`).**
The `authorized_keys` entry points to `/usr/local/bin/conserver-exec` instead
of evaluating `SSH_ORIGINAL_COMMAND` inline. The wrapper validates the IOC
name against a strict regex (`^[a-zA-Z0-9_-]+$`), rejects shell metacharacters,
confirms the UDS socket exists, and only then executes `con -c <path>`.
This eliminates injection risk from crafted SSH commands.

**`con` remains on every node.**
It serves as the execution target for `conserver-exec`. Direct local access
via `ioc-runner attach` continues to work unchanged.

**One conserver daemon manages all nodes.**
All IOC console entries are registered in a single `/etc/conserver.cf` on the
server. Adding or removing IOCs requires only a config regeneration and
a `SIGHUP` to conserver — no restart needed.

**`maxmemb` uses the compile-time default of 16.**
conserver distributes consoles across child processes, each handling up to
`maxmemb` consoles. If a child process crashes (segfault, OOM kill), only
its assigned console sessions are temporarily disconnected — the master
process and all other children remain operational. IOC processes on the
nodes are completely unaffected because the disruption is limited to the
`console` ↔ `conserver` session layer; procServ and the IOC continue running.
Engineers reconnect by re-running `console <ioc-name>`. The default value
of 16 keeps the child count manageable at current scale (~100 IOCs produce
~7 children) while limiting session disruption per child to a small group.

**`timeout` uses the compile-time default of 10 seconds.**
This is the `connect()` timeout for `type exec` SSH connections to nodes.
If a node is unreachable (network failure, host down), conserver waits up
to 10 seconds before reporting the connection as failed. This value balances
responsiveness for the engineer against tolerance for transient network
delays or NFS-heavy environments.

**Compile-time options and rebuild conditions.**
The following options are baked into the conserver binary at compile time.
Changing any of them requires `make conf && make build && make install`
followed by a daemon restart:

| Option                   | Current Value        | Rebuild trigger                          |
|--------------------------|----------------------|------------------------------------------|
| `--with-uds`             | `/run/conserver`     | Change IPC path or switch to TCP         |
| `--with-trust-uds-cred`  | enabled              | Disable peer credential verification     |
| `--with-openssl`         | enabled              | Disable or upgrade OpenSSL linkage       |
| `--with-maxmemb`         | 16 (default)         | Scale to thousands of IOCs               |
| `--with-timeout`         | 10 (default)         | Adjust for high-latency network          |
| `--with-pam`             | disabled             | Add PAM/LDAP authentication              |
| `--with-ipv6`            | disabled             | Enable IPv6 network support              |

Runtime changes (conserver.cf edits, IOC additions/removals, access control
updates) require only `SIGHUP` — no rebuild or restart needed.

**IOC status is on-demand via conserver `task`.**
No polling daemon is introduced. Status is retrieved by firing `^Ec!s` inside
an active console session, which runs a remote SSH command to check
`systemctl is-active` and UDS connection count.

**SSH access uses a dedicated non-login service account.**
A `conserver-svc` account (member of `ioc` group) on each node holds the
SSH public key. The `authorized_keys` entry is locked to the wrapper script,
preventing arbitrary command execution.

**Systemd unit applies defense-in-depth hardening.**
The `conserver.service` unit enforces three layers of protection beyond the
standard daemon lifecycle:

*Service dependencies and preconditions:*
The unit starts only after `network-online.target` (actual network
connectivity, not just interface configuration), `nss-lookup.target`
(DNS resolution for node hostnames), and `time-sync.target` (accurate
log timestamps via NTP). `ConditionPathExists` prevents startup if the
conserver.cf configuration file is missing.

*Security isolation:*
`ProtectSystem=strict` mounts `/`, `/usr`, `/boot` read-only.
`ProtectHome=yes` blocks access to `/home`, `/root`, `/run/user`.
`PrivateTmp=yes` provides an isolated `/tmp` namespace.
`NoNewPrivileges=yes` prevents forked children from gaining additional
privileges. `ReadWritePaths` grants explicit exceptions only for
`/run/conserver` (UDS socket, PID file) and `/var/log/conserver`
(session logs). The configuration file is mounted `ReadOnlyPaths`.

*Resource limits:*
`LimitNOFILE=4096` caps open file descriptors (sufficient for ~2000
console sessions with headroom). `MemoryMax=256M` ensures systemd
intervenes before the OOM killer acts indiscriminately.

---

## 5. Access Control

```
[ ioc group membership ]
     |
     |-- /run/procserv/*/control   (socket mode 0660, owned by ioc-srv:ioc)
     |
     |-- conserver-svc account     (member of ioc group on each node)
           |
           |-- authorized_keys     (forced-command: /usr/local/bin/conserver-exec)
           |-- SSH key source:     /etc/conserver/id_ed25519 (server)

[ Engineer on server ]
     |
     |-- console <ioc-name>        (connects to conserver via UDS /run/conserver)
     |-- conserver access block    (allowed hosts/users defined in conserver.cf)
```

---

## 6. conserver.cf Structure

```
/etc/conserver.cf
  |
  |-- config *          global settings (UDS path, logfile, daemonmode)
  |-- access *          trusted hosts, allowed users
  |-- default epics-ioc shared IOC defaults (rw/ro, logfile, timestamp)
  |-- console <name>    one block per IOC (type exec, SSH command)
  |-- task s            on-demand IOC status (systemctl + UDS connection count)
```

### config block

```
config * {
    defaultaccess allowed;
    logfile /var/log/conserver/conserver.log;
    daemonmode yes;
}
```

The `primaryport` directive is omitted. conserver is compiled with
`--with-uds=/run/conserver`, so client-daemon communication uses the
UNIX domain socket exclusively. No TCP listener is created.

### default block

```
default epics-ioc {
    rw @ioc;
    ro *;
    logfile /var/log/conserver/&.log;
    timestamp 1h;
    options !hupcl;
}
```

### console block (one per IOC)

```
console <ioc-name> {
    master localhost;
    include epics-ioc;
    type exec;
    exec ssh -i /etc/conserver/id_ed25519 \
             -o StrictHostKeyChecking=yes \
             -o BatchMode=yes \
             conserver-svc@<node> <ioc-name>;
}
```

The SSH command passes only the IOC name. The `conserver-exec` wrapper on the
node validates and resolves the full UDS path.

### task block (on-demand status)

```
task s {
    cmd ssh -i /etc/conserver/id_ed25519 \
            -o BatchMode=yes \
            conserver-svc@<node> \
            "systemctl is-active epics-@<ioc-name>.service && \
             ss -lx /run/procserv/<ioc-name>/control 2>/dev/null | \
             awk 'NR>1 {print \"connections:\", \$3}'";
    description IOC service status and UDS connection count;
    confirm no;
}
```

---

## 7. IOC Inventory Source

conserver.cf is generated from the existing `ioc-runner` configuration files.
No separate inventory is maintained.

```
/etc/procServ.d/<ioc-name>.conf
  |
  |-- IOC_PORT  -->  unix:ioc-srv:ioc:0660:/run/procserv/<ioc-name>/control
                                                          ^
                                                          UDS path extracted here
```

The `IOC_PORT` field written by `ioc-runner install` contains the full UDS path
as the last colon-delimited field. The `generate-conserver-cf` script on the
server parses this field to resolve each IOC's socket path.

---

## 8. SSH Key Setup

```
Server
  - Key pair:  /etc/conserver/id_ed25519  (no passphrase)
  - Owner:     conserver:conserver, mode 0600

Each node
  - Account:   conserver-svc  (member of ioc group, no login shell)
  - Wrapper:   /usr/local/bin/conserver-exec
  - authorized_keys entry (restricted):
      restrict,command="/usr/local/bin/conserver-exec" \
      ssh-ed25519 AAAA... conserver@central
```

---

## 9. Repository Structure

```
conserver-env/
├── bin/
│   ├── conserver-exec            # Deployed to each node
│   └── generate-conserver-cf     # Runs on server
├── configure/
│   ├── RELEASE
│   ├── CONFIG, CONFIG_SITE, CONFIG_VARS, CONFIG_SRC
│   ├── CONFIG_SYSTEMD
│   ├── RULES, RULES_FUNC, RULES_PATCH, RULES_SRC
│   ├── RULES_INSTALL             # conserver build targets
│   ├── RULES_SYSTEMD             # server daemon targets (sd_*)
│   └── RULES_DEPLOY              # server/node deployment targets
├── site-template/
│   └── conserver.service.in
└── docs/
    ├── ACCOUNT.md
    └── ARCHITECTURE.md
```

### Makefile Targets

| Scope      | Target            | Description                                    |
|------------|-------------------|------------------------------------------------|
| Build      | `make init`       | Clone conserver source                         |
| Build      | `make conf`       | Configure with OpenSSL                         |
| Build      | `make build`      | Compile conserver                              |
| Build      | `make install`    | Install to `/opt/conserver`                    |
| Server     | `make sd_useradd` | Create `conserver` system account              |
| Server     | `make sd_install` | Deploy systemd unit + daemon-reload            |
| Server     | `make sd_enable`  | Enable conserver on boot                       |
| Server     | `make sd_start`   | Start conserver daemon                         |
| Node       | `make node_useradd`   | Create `conserver-svc` account (ioc group) |
| Node       | `make node_install`   | Deploy `conserver-exec` to `/usr/local/bin`|
| Node       | `make node_uninstall` | Remove `conserver-exec`                    |
| Node       | `make node_userdel`   | Remove `conserver-svc` account             |

---

## 10. conserver.cf Lifecycle

```
IOC added    -->  ioc-runner install <name>.conf   (on node)
                  -->  /etc/procServ.d/<name>.conf written

Config regen -->  generate-conserver-cf            (on server)
                  -->  /etc/conserver.cf.new

Validate     -->  conserver -t -C /etc/conserver.cf.new

Apply        -->  mv /etc/conserver.cf.new /etc/conserver.cf
                  kill -HUP $(pidof conserver)     (no restart required)
```

---

## 11. Deployment Phases

```
Phase 1. Local test environment
         - 2x VMs (Debian 13 or Rocky 8.10)
         - alsu-rocky8-gui-test : server (conserver)
         - alsu-rocky8-test     : node (procServ + ioc-runner + con)

Phase 2. Node setup
         - make node_useradd     (create conserver-svc on node)
         - make node_install     (deploy conserver-exec on node)

Phase 3. SSH key setup
         - Generate key on server
         - Deploy public key to conserver-svc authorized_keys on node
         - Verify: ssh conserver-svc@<node> <ioc-name>

Phase 4. Server setup
         - make init && make conf && make build && make install
         - make sd_useradd && make sd_install && make sd_enable
         - generate-conserver-cf
         - conserver -t
         - make sd_start

Phase 5. Validation
         - console <ioc-name>    attach from server
         - ^Ec!s                 status task
         - console -i            list all consoles
         - Two concurrent sessions on same IOC

Phase 6. Production rollout
         - Deploy conserver-svc + conserver-exec to all nodes
         - Regenerate conserver.cf for full IOC inventory
         - Validate and reload
```

---

## 12. Validation Checklist

```
[ ] procServ + con + ioc-runner operational on node
[ ] UDS confirmed: ss -lx /run/procserv/<ioc-name>/control
[ ] conserver-svc account exists on node, member of ioc group
[ ] conserver-exec installed: /usr/local/bin/conserver-exec
[ ] SSH key generated on server (conserver:conserver, mode 0600)
[ ] SSH public key deployed to node authorized_keys
[ ] SSH exec verified: ssh conserver-svc@<node> <ioc-name>
[ ] conserver built and installed on server
[ ] conserver.cf generated and validated: conserver -t
[ ] conserver daemon started: systemctl start conserver
[ ] console <ioc-name> attaches from server
[ ] ^Ec!s returns systemctl status and connection count
[ ] Two concurrent console sessions verified
[ ] /var/log/conserver/<ioc-name>.log written
[ ] IOC restart: console session recovers automatically
```
