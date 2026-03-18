# conserver Configuration Environment

`conserver` build environment for Linux.

* The source codes are located at <https://github.com/bstansell/conserver>

## About

This repository builds the `conserver` package consistently on Linux
from a pinned source tag.

## Prerequisites

```bash
# Rocky Linux 8
sudo dnf install autoconf automake gcc make openssl-devel

# Debian / Ubuntu
sudo apt install autoconf automake gcc make libssl-dev
```

## Commands

By default, everything installs under `/usr/local`. Override via:

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
make uninstall
make clean
make distclean
```
