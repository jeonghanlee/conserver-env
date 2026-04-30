# conserver Configuration Environment

`conserver` build environment for Linux.

* Source: <https://github.com/bstansell/conserver>
* See [docs/ACCOUNT.md](docs/ACCOUNT.md) for service account setup.

## Prerequisites

```bash
# Rocky Linux 8
sudo dnf install autoconf automake gcc make openssl-devel

# Debian / Ubuntu
sudo apt install autoconf automake gcc make libssl-dev
```

## Commands

Default install path is `/opt/conserver`. Override via:

```bash
echo "INSTALL_LOCATION=where...." > configure/CONFIG_SITE.local
```

```bash
make init
make conf
make build
make install
```

```bash
make sd_useradd
make sd_install
make sd_enable
make sd_start
```

Configuration deploy and apply (5-stage workflow, see [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)):

```bash
make cf_deploy        # cf_check + cf_install
make cf_apply         # cf_check_installed + cf_reload
```

```bash
make sd_stop
make sd_disable
make sd_clean
make sd_userdel
make uninstall
make distclean
```
