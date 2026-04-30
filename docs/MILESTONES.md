# conserver-env Development Milestones

## M1. Documentation and Design

- [x] ARCHITECTURE.md — server/node terminology, wrapper script, bin/ structure
- [x] ACCOUNT.md — server account, node account, SSH key setup, reload workflow
- [x] `bin/conserver-exec` — SSH forced-command wrapper with strict IOC name validation
- [x] ioc-runner `system-wide/README.md` — conserver-env repository link
- [x] UDS client-daemon design decision (TCP 7782 eliminated)
- [x] Compile-time options table and rebuild conditions
- [x] systemd unit hardening (security isolation, resource limits, dependencies)
- [x] Makefile infrastructure (CONFIG_DEPLOY, RULES_DEPLOY, CONFIG, RULES)
- [x] Inventory model: split-per-node `#include`, git-managed (no scraping)
- [x] `conserver.cf.in` template + `@INSTALL_LOCATION@` rendering
- [x] CONFIG_CF + RULES_CF — 5-stage cf workflow + `cf_deploy`/`cf_apply` groups
- [x] Test inventory committed: `testbed-rocky8-node{1,2}.cf` (3 IOCs each)

## M2. Build

- [x] `make init` on Debian 13
- [x] `make conf` on Debian 13 (--with-openssl, --with-uds, --with-trust-uds-cred)
- [x] `make build` on Debian 13
- [x] `make install` on Debian 13 (INSTALL_LOCATION=/usr/local)
- [ ] `make init && make conf && make build && make install` on Rocky 8.10

## M3. Server Setup

- [x] `make sd_useradd` — conserver account created
- [x] `make sd_conf && make sd_conf.show` — unit file verified
- [x] `make sd_install` — unit deployed to /etc/systemd/system
- [x] `LogsDirectory=conserver` — systemd auto-creates `/var/log/conserver` 0750
- [x] `make cf_conf` — `conserver.cf.in` -> `conserver.cf` rendering
- [ ] `make cf_deploy` (cf_check + cf_install) on testbed
- [ ] `make sd_enable`
- [ ] `make sd_start` — conserver daemon running on UDS /run/conserver
- [ ] `make cf_apply` (cf_check_installed + cf_reload)
- [x] `task s` block design: `subst ~=hs,^=cs`, `host` directive added to each console block, `make cf_check` passes

## M4. Node Setup

- [ ] `make node_useradd` — conserver-svc account created on node
- [ ] `make node_install` — conserver-exec deployed to `/usr/local/bin`
- [ ] `make node_uninstall` — verified removal
- [ ] `make node_userdel` — verified removal

## M5. SSH Integration

- [ ] SSH key pair generated on server (`/etc/conserver/id_ed25519`)
- [ ] Public key deployed to node `authorized_keys` with `restrict` + forced-command
- [ ] sshd configuration verified on node
- [ ] `ssh conserver-svc@<node> <ioc-name>` triggers `conserver-exec` → `con` → UDS

## M6. End-to-End Validation

- [ ] `console <ioc-name>` attaches from server through full chain
- [ ] `^Ec!s` task returns systemctl status and connection count
- [ ] Two concurrent sessions on same IOC
- [ ] IOC restart: console session recovers automatically
- [ ] `/var/log/conserver/<ioc-name>.log` written

## M7. Production Rollout

- [ ] `conserver-svc` + `conserver-exec` deployed to all production nodes
- [ ] Site inventory authored in site git repo (cf.in + per-node cf files)
- [ ] `make cf_deploy && make cf_apply` against production cf
- [ ] Operations documentation finalized
