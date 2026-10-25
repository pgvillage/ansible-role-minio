# pgvillage.minio – Role API

This document describes all variables that can be set to configure the `pgvillage.minio` role.
Defaults are defined in [`defaults/main.yml`](../defaults/main.yml).

The role:

1. optionally asserts that certificates are set up (`minio_cert_managed`),
2. creates the Minio user, group and data directories, installs the packages and generates the server environment file and service definition (systemd or init.d),
3. deploys TLS certificates for the server and client,
4. starts Minio and waits until the endpoint responds,
5. configures the Minio client (`mcli`) and creates buckets.

With `minio_use_rustfs: true` it replaces Minio by RustFS instead; see [RustFS](#rustfs).

## Installation

| Variable | Default | Description |
|----------|---------|-------------|
| `minio_package_state` | `present` | State of the Minio packages and the Minio group (`present` or `absent`). |
| `minio_package_names` | `[minio, mcli]` | Packages to install: the Minio server and the Minio client (mcli). |
| `minio_server_bin` | `/usr/local/bin/minio` | Path to the Minio server binary, used by the systemd unit and the init.d script. |
| `minio_client_bin` | `/usr/local/bin/mcli` | Path to the Minio client binary, used to create buckets. |
| `minio_service_name` | `minio` | Name of the Minio server service (systemd unit or init.d script). |
| `minio_user` | `minio` | User that runs the Minio server service. |
| `minio_group` | `{{ minio_user }}` | Group of the Minio server service. |

## RustFS

Setting `minio_use_rustfs` to `true` replaces Minio by [RustFS](https://rustfs.com). The role then stops and removes Minio, **leaving the data directories untouched**, and installs and configures RustFS with the same user, data directories, listen and console addresses, credentials and certificates.

> **Back up the data first.** RustFS reads the Minio data in place (unencrypted objects; this role does not configure encryption) and imports the bucket settings and users from `.minio.sys` into its own `.rustfs.sys`. The switch is one-way: Minio only reads `.minio.sys`, so it would not see buckets, users or policies created or changed under RustFS. See [MinIO on-disk format interoperability](https://github.com/rustfs/rustfs/blob/main/docs/architecture/minio-file-format-compat.md).

With `minio_use_rustfs: true` the role:

1. creates the user, group and data directories, as for Minio;
2. stops and disables the Minio service, and removes its systemd unit or init.d script, the env file and the packages in `minio_package_names`. The Minio user, data directories, certificates and client config are kept;
3. installs the RustFS packages and generates the RustFS env file and systemd unit. RustFS needs systemd: the role fails on hosts with another service manager;
4. deploys the certificates: `minio_cert_files` plus `minio_rustfs_cert_files`;
5. starts RustFS and waits until the endpoint responds (the same check as for Minio);
6. configures the RustFS client (`rc`) with a `minio` alias and creates `minio_buckets` with it.

| Variable | Default | Description |
|----------|---------|-------------|
| `minio_use_rustfs` | `false` | Replace Minio by RustFS. |
| `minio_rustfs_package_names` | `[rustfs, rc]` | Packages to install: the RustFS server and the RustFS client (rc). Upstream names the client package `rustfs-cli`. |
| `minio_rustfs_server_bin` | `/usr/bin/rustfs` | Path to the RustFS server binary, used by the systemd unit. |
| `minio_rustfs_client_bin` | `/usr/bin/rc` | Path to the RustFS client binary, used to create buckets. |
| `minio_rustfs_service_name` | `rustfs` | Name of the RustFS server service. The role writes `/etc/systemd/system/<name>.service`, which overrides the unit shipped in the package. It runs as `minio_user`. For ports below 1024 (`minio_server_addr`, `minio_server_console_addr`), it gets `CAP_NET_BIND_SERVICE`. |
| `minio_rustfs_server_envfile` | `/etc/default/rustfs` | Path to the file containing the environment variables for the RustFS server. Sets `RUSTFS_VOLUMES` (like `MINIO_VOLUMES`), `RUSTFS_ADDRESS` (`minio_server_addr`), `RUSTFS_CONSOLE_ADDRESS` (`minio_server_console_addr`), `RUSTFS_TLS_PATH` (`minio_rustfs_tls_path`) and `RUSTFS_ACCESS_KEY` / `RUSTFS_SECRET_KEY` (`minio_access_key` / `minio_secret_key`). `minio_server_opts` and `minio_server_env_extra` do not apply to RustFS: use `minio_rustfs_env_extra`. |
| `minio_rustfs_env_extra` | `""` | Extra environment variables, added verbatim to the RustFS env file (e.g. `RUSTFS_REGION=eu-west-1`). |
| `minio_rustfs_tls_path` | `<minio home>/.minio/certs` | Directory with the RustFS server certificate and key (`rustfs_cert.pem` and `rustfs_key.pem`), set as `RUSTFS_TLS_PATH`. |
| `minio_rustfs_client_configdir` | `<minio home>/.config/rc` | Config directory of the RustFS client. The role writes `config.toml` with a `minio` alias from the same endpoint config as mcli (`minio_endpoint`): URL, keys, bucket lookup (`path`), and `insecure` from `minio_insecure`. |
| `minio_rustfs_client_ca_bundle` | the `minio_chain` path when `minio_cert_from_string`, else `""` | CA bundle (PEM) rc uses to verify the server certificate. Empty means the system CAs. |
| `minio_rustfs_cli_options` | `--ignore-existing` | Options added to rc commands, such as when creating buckets. |

## Server configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `minio_server_envfile` | `/etc/default/minio` | Path to the file containing the environment variables for the Minio server. |
| `minio_server_addr` | `:9091` | Listen address (`[host]:port`). For ports below 1024, the systemd service gets `CAP_NET_BIND_SERVICE`. |
| `minio_server_console_addr` | `:9001` | Console (web UI) listen address (`[host]:port`), passed as `--console-address`. Also used for the RustFS console. For ports below 1024, the systemd service gets `CAP_NET_BIND_SERVICE`. |
| `minio_server_datadirs` | `[/var/lib/minio]` | Data directories. The role creates them, and uses them as `MINIO_VOLUMES` when `minio_server_cluster_nodes` is empty. |
| `minio_server_cluster_nodes` | `[]` | Cluster node list (e.g. `https://minio{1...4}/var/lib/minio`). When set, used as `MINIO_VOLUMES` instead of `minio_server_datadirs`. |
| `minio_server_env_extra` | `""` | Extra environment variables, added verbatim to the env file (e.g. `MINIO_BROWSER=off`). |
| `minio_server_opts` | `""` | Additional Minio server CLI options, appended to `MINIO_OPTS`. |
| `minio_server_configdir` | `<minio home>/.minio` | Config directory of the Minio server. Server certificates are stored in `certs/` under this directory. |

## Credentials

| Variable | Default | Description |
|----------|---------|-------------|
| `minio_access_key` | `AWSKEY` | Access key. Set as `MINIO_ROOT_USER` for the server and used by the client. **Override this.** |
| `minio_secret_key` | `AWSSECRET` | Secret key. Set as `MINIO_ROOT_PASSWORD` for the server and used by the client. **Override this.** |

## Client configuration and buckets

| Variable | Default | Description |
|----------|---------|-------------|
| `minio_endpoint` | `{}` | Overrides for the client endpoint config. Merged on top of the internal defaults below. The result is used for the `minio` mcli alias and for the startup health check. |
| `minio_buckets` | `[mybucket]` | Buckets to create with mcli after Minio has started. |
| `minio_client_configdir` | `<minio home>/.mcli` | Config directory of the Minio client (`config.json` and trusted CAs). |
| `minio_insecure` | `false` | Pass `--insecure` to mcli, which skips TLS certificate verification. |
| `minio_cli_options` | `--ignore-existing` (+ `--insecure`) | Options added to mcli commands, such as when creating buckets. |

The internal variable `__minio_endpoint_defaults` holds these defaults. Override them through `minio_endpoint`:

| Key | Default |
|-----|---------|
| `accessKey` | `{{ minio_access_key }}` |
| `secretKey` | `{{ minio_secret_key }}` |
| `url` | `https://{{ ansible_facts.fqdn }}` |
| `api` | `S3v4` |
| `path` | `auto` |

Example:

```yaml
minio_endpoint:
  url: "https://minio.example.com:9091"
```

## Certificates

| Variable | Default | Description |
|----------|---------|-------------|
| `minio_cert_managed` | `true` | Whether certificates are managed (e.g. by [chainsmith](https://github.com/MannemSolutions/chainsmith)). When true, the role asserts that `minio_client_cert`, `minio_server_cert`, `minio_server_key` and `minio_server_chain` contain real (multi-line) PEM data. |
| `minio_cert_from_string` | `{{ minio_cert_managed }}` | Deploy certificates from the `body` strings in `minio_cert_files` (true), or copy them from the `source` files (false). |
| `minio_cert_remote` | `{{ not minio_cert_from_string }}` | When copying from file: `source` is on the managed host (true) or on the Ansible controller (false). |
| `minio_client_cert` | `---- CERT ----` | CA chain (PEM) trusted by the Minio client. |
| `minio_server_cert` | `---- CERT ----` | Server certificate (PEM). |
| `minio_server_key` | `---- KEY ----` | Server private key (PEM). |
| `minio_server_chain` | `---- CHAIN ----` | Server CA chain (PEM). |
| `minio_cert_folders` | see below | Directories to create for certificates. |
| `minio_cert_files` | see below | Certificate files to deploy. |

### `minio_cert_folders`

A dict of directories. Each value has a `path` and an `owner`. The directories are created with mode `0700`.

```yaml
minio_cert_folders:
  client:
    path: "{{ minio_client_configdir }}/certs/CAs/"
    owner: "{{ minio_user }}"
  server:
    path: "{{ minio_server_configdir }}/certs"
    owner: "{{ minio_user }}"
```

### `minio_cert_files`

A dict of certificate files. Each file is deployed with mode `0600`. Changes notify the `Restart minio` handler (it restarts RustFS when `minio_use_rustfs` is true); multiple notifications are combined into one restart per handler flush. The role flushes handlers before its startup check. Each value has these keys:

| Key | Description |
|-----|-------------|
| `path` | Destination path. |
| `owner` | File owner (default `root`). |
| `body` | Content, used when `minio_cert_from_string` is true. |
| `source` | Source file, used when `minio_cert_from_string` is false. Entries without a `source` are skipped. |

Defaults:

| Entry | Path | Body | Source |
|-------|------|------|--------|
| `minio_chain` | `<client certs>/CA_<hostname>.crt` | `minio_client_cert` | – |
| `server_cert` | `<server certs>/public.crt` | `minio_server_cert` | `/etc/pki/minio/server.crt` |
| `server_key` | `<server certs>/private.key` | `minio_server_key` | `/etc/pki/minio/server.key` |
| `server_chain` | `<server certs>/ca.pem` | `minio_server_chain` | `/etc/pki/minio/root.crt` |

### `minio_rustfs_cert_files`

With `minio_use_rustfs: true`, these files are deployed as well, in the same way as `minio_cert_files` (same keys). The directory `minio_rustfs_tls_path` is created like the `minio_cert_folders`.

| Entry | Path | Body | Source |
|-------|------|------|--------|
| `rustfs_cert` | `<minio_rustfs_tls_path>/rustfs_cert.pem` | `minio_server_cert` | `/etc/pki/minio/server.crt` |
| `rustfs_key` | `<minio_rustfs_tls_path>/rustfs_key.pem` | `minio_server_key` | `/etc/pki/minio/server.key` |

## Example

```yaml
- hosts: minio
  roles:
    - role: pgvillage.minio
      vars:
        minio_access_key: "{{ vault_minio_access_key }}"
        minio_secret_key: "{{ vault_minio_secret_key }}"
        minio_buckets:
          - backups
        minio_cert_managed: false
        minio_cert_remote: true
```
