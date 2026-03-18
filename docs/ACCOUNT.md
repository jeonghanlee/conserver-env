# conserver Service Account and Group

## Account Layout

| Attribute    | Value                        |
|--------------|------------------------------|
| User         | `conserver`                  |
| Group        | `conserver`                  |
| Home         | `/nonexistent`               |
| Shell        | `/sbin/nologin`              |
| Purpose      | conserver daemon on the central server only |

The `conserver` account is a system account with no login shell and no home
directory. It exists solely to run the conserver daemon process under a
restricted identity.

## Scope

This account is required only on the **central server** where the conserver
daemon runs. IOC servers do not require this account.

## Manual Setup

```bash
groupadd conserver
useradd -r -M -d /nonexistent -g conserver -s /sbin/nologin \
        -c "conserver daemon account" conserver
```

## Installation Path Ownership

The conserver binaries are installed under `/opt/conserver`. The daemon
process runs as `conserver:conserver` and requires read and execute
permissions on this path.

```bash
sudo chown -R root:conserver /opt/conserver
sudo chmod -R 755 /opt/conserver
```

## systemd Service Identity

The systemd unit file specifies:

```ini
[Service]
User=conserver
Group=conserver
```

This ensures the daemon process runs with minimum privilege. No sudo or
setuid configuration is required for normal operation.
