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
- [ ] `bin/generate-conserver-cf` script written
- [ ] Parses `/etc/procServ.d/*.conf` and extracts IOC_PORT UDS path
- [ ] Node inventory collection method finalized (SSH pull or NFS)
- [ ] Generates valid `conserver.cf` with config, access, default, console blocks
- [ ] `/var/log/conserver/` directory created with proper ownership
- [ ] `conserver -t -C /usr/local/etc/conserver.cf` validation passes
- [ ] `make sd_enable`
- [ ] `make sd_start` — conserver daemon running on UDS /run/conserver

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
- [ ] `conserver.cf` generated for full IOC inventory
- [ ] conserver daemon reloaded with production configuration
- [ ] Operations documentation finalized
