# EPICS IOC conserver Integration Architecture

## 1. Overview

This document describes the architecture for integrating `conserver` with the
`epics-ioc-runner` management environment. The goal is to provide centralized,
multi-user console access to EPICS IOCs running under `procServ` across multiple
nodes, without modifying the existing `ioc-runner` deployment model.

**Terminology:**

| Term              | Description                                              |
|-------------------|----------------------------------------------------------|
| **server**        | Central server running the conserver daemon + `console`  |
| **node** (IOC server) | Host running procServ + ioc-runner + con             |

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
  - cf inventory: /usr/local/etc/conserver.cf + conserver.d/<vm-name>.cf
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
All IOC console entries are registered in a single `/usr/local/etc/conserver.cf` on the
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

The configuration is split across one main file plus one file per node,
linked via the conserver `#include` directive (manpage `conserver.cf(5)`,
column 0, max 10 levels of nesting):

```
/usr/local/etc/conserver.cf                       (main: globals + #include lines)
  |
  |-- config *          global settings (UDS path, logfile, daemonmode)
  |-- access *          trusted hosts, allowed users
  |-- default epics-ioc shared IOC defaults (master, type, rw/ro, logfile, timestamp)
  |-- #include /usr/local/etc/conserver.d/<vm-name>.cf  (one per node)
  |-- task s            on-demand IOC status (systemctl + UDS connection count)

/usr/local/etc/conserver.d/<vm-name>.cf           (per-node: console blocks)
  |
  |-- console <ioc-name> { include epics-ioc; exec ssh ... ; }   (one per IOC)
  |-- ...
```

The main file is rendered from a template (`site-template/etc/conserver.cf.in`)
with `@INSTALL_LOCATION@` substituted. Per-node files are committed as-is.

### config block (main)

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

### default block (main)

```
default epics-ioc {
    rw @ioc;
    ro *;
    logfile /var/log/conserver/&.log;
    timestamp 1h;
    options !hupcl;
    master localhost;
    type exec;
}
```

`master` and `type` are pushed into the default block so each per-node
console block only carries the unique `exec` SSH command.

### console block (one per IOC, in per-node file)

```
console <ioc-name> {
    include epics-ioc;
    exec ssh -i /etc/conserver/id_ed25519 -o StrictHostKeyChecking=yes -o BatchMode=yes conserver-svc@<vm-name> <ioc-name>;
}
```

The SSH command passes only the IOC name. The `conserver-exec` wrapper on the
node validates and resolves the full UDS path.

### task block (main)

The status task uses the conserver `subst` directive to substitute the
console name (`c`) and host (`h`) values at invocation time. The exact
`subst` design is pending; see MILESTONES for the open item.

---

## 7. IOC Inventory Source

The IOC inventory is maintained as version-controlled config files in the
git repository, not generated from `/etc/procServ.d/` scraping. Each node
gets one file under `site-template/etc/conserver.d/<vm-name>.cf` containing
that node's console blocks.

```
git repository (this conserver-env repo or a downstream site repo)
  |
  |-- site-template/etc/conserver.cf.in              (main, with @INSTALL_LOCATION@)
  |-- site-template/etc/conserver.d/<vm-name>.cf     (per-node console blocks)
```

Workflow:

```
1. Operator edits cf.in or per-node files in repo (git branch + PR)
2. `make cf_deploy`   -- syntax check (-S in staging) + install to /usr/local/etc/
3. `make cf_apply`    -- syntax check installed copy + systemctl reload (SIGHUP)
```

Adding/removing a node = adding/removing one file under `conserver.d/` and
updating the corresponding `#include` line in `conserver.cf.in`. Adding/
removing IOCs on an existing node = editing that node's per-node file.
git history records every change.

The site-specific production inventory lives in the site's internal git
repository. This `conserver-env` repository carries a test inventory
(`testbed-rocky8-node1.cf`, `testbed-rocky8-node2.cf`) for cloud-provision
integration testing.

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
│   └── conserver-exec                       # Deployed to each node
├── configure/
│   ├── RELEASE
│   ├── CONFIG, CONFIG_SITE, CONFIG_VARS, CONFIG_SRC
│   ├── CONFIG_SYSTEMD                       # conserver/conserver-svc account vars
│   ├── CONFIG_DEPLOY                        # node-side install vars
│   ├── CONFIG_CF                            # conserver.cf path/staging vars
│   ├── RULES, RULES_FUNC, RULES_PATCH, RULES_SRC
│   ├── RULES_INSTALL                        # conserver build targets
│   ├── RULES_SYSTEMD                        # server daemon targets (sd_*)
│   ├── RULES_DEPLOY                         # node deployment targets (node_*)
│   ├── RULES_CF                             # cf install/check/reload targets (cf_*)
│   └── RULES_VARS
├── site-template/
│   ├── systemd/
│   │   ├── conserver.service.in             # Templated systemd unit
│   │   └── conserver.service                # Rendered output
│   └── etc/
│       ├── conserver.cf.in                  # Templated main cf
│       ├── conserver.cf                     # Rendered output
│       └── conserver.d/
│           ├── testbed-rocky8-node1.cf      # Test inventory: per-node IOCs
│           └── testbed-rocky8-node2.cf
└── docs/
    ├── ACCOUNT.md
    ├── ARCHITECTURE.md
    └── MILESTONES.md
```

### Makefile Targets

| Scope      | Target                      | Description                                                       |
|------------|-----------------------------|-------------------------------------------------------------------|
| Build      | `make init`                 | Clone conserver source                                            |
| Build      | `make conf`                 | Configure with OpenSSL + UDS                                      |
| Build      | `make build`                | Compile conserver                                                 |
| Build      | `make install`              | Install to `$(INSTALL_LOCATION)`                                  |
| Server     | `make sd_useradd`           | Create `conserver` system account                                 |
| Server     | `make sd_install`           | Deploy systemd unit + daemon-reload                               |
| Server     | `make sd_enable`            | Enable conserver on boot                                          |
| Server     | `make sd_start`             | Start conserver daemon                                            |
| Node       | `make node_useradd`         | Create `conserver-svc` account (ioc group)                        |
| Node       | `make node_install`         | Deploy `conserver-exec` to `/usr/local/bin`                       |
| Node       | `make node_uninstall`       | Remove `conserver-exec`                                           |
| Node       | `make node_userdel`         | Remove `conserver-svc` account                                    |
| Config     | `make cf_conf`              | Render `conserver.cf.in` -> `conserver.cf`                        |
| Config     | `make cf_check`             | Stage 2 -- staging render + `conserver -S` syntax check           |
| Config     | `make cf_install`           | Stage 3 -- install cf and conserver.d/ to `$(INSTALL_LOCATION)/etc/` |
| Config     | `make cf_check_installed`   | Stage 4 -- syntax check the installed cf                          |
| Config     | `make cf_reload`            | Stage 5 -- `systemctl reload conserver` (SIGHUP)                  |
| Config     | `make cf_deploy`            | Group: `cf_check` then `cf_install`                               |
| Config     | `make cf_apply`             | Group: `cf_check_installed` then `cf_reload`                      |

---

## 10. conserver.cf Lifecycle

The 5-stage operator workflow is exposed through `cf_*` Makefile targets.
Stages 2 and 3 are bundled by `cf_deploy`; stages 4 and 5 by `cf_apply`.

```
Stage 1. Manual edit                                      (no make target)
         - Operator edits site-template/etc/conserver.cf.in
           or site-template/etc/conserver.d/<vm-name>.cf
         - git commit + push (audit trail)

Stage 2. Syntax check                                     make cf_check
         - Render cf.in to $(TOP)/build/etc/ with #include
           paths pointing to the staging dir
         - Copy per-node files to staging
         - conserver -S -C build/etc/conserver.cf

Stage 3. Install                                          make cf_install
         - cf_conf renders cf.in with INSTALL_LOCATION
         - Manifest-based stale cleanup of /usr/local/etc/conserver.d/
         - Install rendered cf and per-node files (root:conserver, 0640)
         - Write new manifest of installed per-node files

Stage 4. Production syntax check                          make cf_check_installed
         - conserver -S -C /usr/local/etc/conserver.cf

Stage 5. Reload                                           make cf_reload
         - systemctl reload conserver  (= SIGHUP via ExecReload)
         - Daemon re-reads cf without restart;
           existing console sessions unaffected
```

`make cf_deploy` runs stages 2 and 3 in order. `make cf_apply` runs stages
4 and 5 in order. If `cf_check` fails, `cf_install` is not executed; if
`cf_check_installed` fails, `cf_reload` is not executed.

---

## 11. Deployment Phases

```
Phase 1. Local test environment (cloud-provision)
         - 1 server + 2 node VMs per OS, naming testbed-<os>-{server,node1,node2}
         - testbed-rocky8-server : server (conserver)
         - testbed-rocky8-node1, node2 : nodes (procServ + ioc-runner + con)

Phase 2. Node setup (on each node VM)
         - make node_useradd     (create conserver-svc, member of ioc group)
         - make node_install     (deploy conserver-exec to /usr/local/bin)

Phase 3. SSH key setup
         - Generate key on server (conserver:conserver, mode 0600)
         - Deploy public key to conserver-svc authorized_keys on each node
         - Verify: ssh conserver-svc@<vm-name> <ioc-name>

Phase 4. Server setup (on server VM)
         - make init && make conf && make build && make install
         - make sd_useradd && make sd_install && make sd_enable
         - make cf_deploy        (cf_check + cf_install)
         - make sd_start
         - make cf_apply         (cf_check_installed + cf_reload)

Phase 5. Validation
         - console <ioc-name>    attach from server
         - ^Ec!s                 status task
         - console -i            list all consoles
         - Two concurrent sessions on same IOC

Phase 6. Production rollout
         - Deploy conserver-svc + conserver-exec to all production nodes
         - Author site inventory (cf.in + per-node cf files) in site git repo
         - make cf_deploy && make cf_apply
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
[ ] cf authored in repo (cf.in + conserver.d/<vm-name>.cf)
[ ] make cf_deploy passes (cf_check + cf_install)
[ ] conserver daemon started: systemctl start conserver
[ ] make cf_apply passes (cf_check_installed + cf_reload)
[ ] console <ioc-name> attaches from server
[ ] ^Ec!s returns systemctl status and connection count
[ ] Two concurrent console sessions verified
[ ] /var/log/conserver/<ioc-name>.log written
[ ] IOC restart: console session recovers automatically
```
