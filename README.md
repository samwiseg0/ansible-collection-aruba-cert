# samwiseg0.aruba_cert

This collection pushes a TLS server certificate to an Aruba Mobility Conductor cluster that runs
ArubaOS 8.

See the [CHANGELOG](CHANGELOG.md) for the release notes. Report problems on the
[issues page](https://github.com/samwiseg0/ansible-collection-aruba-cert/issues).

## What it does

The `server_cert` role runs on the control node. It works on each config path you list.

1. It discovers the cluster nodes from the conductor. It stops if a path has no nodes, or if a
   node of a path is missing from `show switches`.
2. It reads the current binds over the REST API and reports each path as `current` or
   `would push`.
3. With `server_cert_apply: true`, it also writes to the cluster.
   - It stops if the cluster has not converged.
   - It builds a PKCS12 file (p12), and the conductor pulls the p12 over scp.
   - It imports the certificate on the conductor and checks the import there. It then registers
     and binds the certificate. Each of these two steps has its own write memory and a check on
     every node.
   - On each path that binds `switch-cert`, it checks that every node serves the new
     certificate. A node normally restarts httpd by itself. The role forces a restart only on a
     node that serves an old certificate and whose config binds the current name.
   - It deletes old unbound certificates that carry your name prefix. It first checks every
     node for a reference to the name.

The certificate name is `<prefix>_<YYYYMMDD>_<suffix>`. The date is the notAfter date of the
certificate in UTC. A run with the same certificate changes nothing. Two certificates with the
same notAfter date get the same name. The report flags the second one, and an apply run stops
before any change. To push it, change the suffix.

An httpd restart drops the web UI and the captive portal on that node for about 20 seconds.
Silence your monitoring for the cluster before an apply run.

The role has no lock. Run one instance at a time against a cluster.

Each push leaves the encrypted p12 on the flash of the conductor. The role uses a random
passphrase and discards it after the run. To remove an old p12, run `delete filename <file>` on
the conductor.

The role does not issue certificates. Give it PEM files from any source.

## Requirements

- ansible-core 2.19 or later.
- The control node needs bash, python3, openssl, sshpass and the `timeout` command from
  coreutils.
- The management user needs ssh access on the conductor and on every node. It needs REST API
  access on the conductor. The same user and password must work on every node.
- The control node reaches the conductor and every node on tcp/22 and on `server_cert_port`.
- The conductor reaches `server_cert_scp_host` on tcp/22.
- Your scp mode has its own needs. See below.

## Install

```bash
ansible-galaxy collection install git+https://github.com/samwiseg0/ansible-collection-aruba-cert.git,v1.0.0
```

You can also list the collection in a `requirements.yml` file.

```yaml
collections:
  - name: https://github.com/samwiseg0/ansible-collection-aruba-cert.git
    type: git
    version: v1.0.0
```

Then run `ansible-galaxy collection install -r requirements.yml`. Both forms need git on the
machine that installs the collection.

## Quick start

Put your settings in a file, for example `aruba_cert.yml`. It holds passwords, so encrypt it with
`ansible-vault encrypt aruba_cert.yml`.

```yaml
server_cert_conductor: conductor.example.com
server_cert_username: api-admin
server_cert_password: change-me
server_cert_fullchain: /etc/ssl/example/fullchain.pem
server_cert_privkey: /etc/ssl/example/privkey.pem
server_cert_name_prefix: EXAMPLE
server_cert_scp_host: 192.0.2.10
server_cert_paths:
  - path: /mm
    suffix: MM
    bind_fields: [switch-cert]
  - path: /md/CAMPUS
    suffix: MD
    bind_fields: [switch-cert, captive-portal-cert]
```

This command only reports.

```bash
ansible-playbook samwiseg0.aruba_cert.server_cert -e @aruba_cert.yml --ask-vault-pass
```

This command writes to the cluster. The temp account mode needs become on the control node, so
the command has `-K`.

```bash
ansible-playbook samwiseg0.aruba_cert.server_cert -e @aruba_cert.yml -e server_cert_apply=true --ask-vault-pass -K
```

A run with `--check` only reports, even with `server_cert_apply=true`.

The first run against a factory certificate needs `-e server_cert_validate_certs=false`.

To use the role in your own play, run it on `localhost` with `gather_facts: false`.

## scp modes

The conductor pulls the p12 with `copy scp:`. The role supports two modes.

**Temp account (default).** Leave `server_cert_scp_user` empty. For each upload, the role creates
a system account on the control node with a random password. The account name is
`server_cert_temp_user` (default `certdrop`). The role puts the p12 in the account home. It
deletes the account and its home after the upload. The role stops if the account, a group with its
name or its home exists before the upload.

- The role needs root on the control node (become).
- sshd on the control node must accept password logins for the temp account. No `AllowUsers`,
  `AllowGroups`, `DenyUsers` or `Match` rule may block it for the conductor addresses.
- `server_cert_scp_host` is the control node address that the conductor can reach.

You can limit the temp account to the conductor addresses. Put this block at the end of
`sshd_config`. Replace the addresses with the addresses of your conductor nodes.

```text
Match User certdrop Address *,!192.0.2.21,!192.0.2.22
    DenyUsers certdrop
```

**Remote scp host.** Set `server_cert_scp_user`, `server_cert_scp_password` and an absolute
`server_cert_scp_dir`. The directory must exist on the scp host. For each upload, the role copies
the p12 into `server_cert_scp_dir` on the scp host. The copy has mode 0600 and owner
`server_cert_scp_user`. The role connects to `server_cert_scp_delegate`, or to
`server_cert_scp_host` when the delegate is empty. The scp host needs Python for the copy and file
modules. The role deletes the copy after the upload. The role builds the p12 in a local temp
directory and deletes the directory at the end of the push.

- This mode needs no root on the control node.
- Your connection user for the scp host can differ from the scp user. In that case, set
  `ansible_become` for the scp host in your inventory.

## Host keys and TLS

With an empty `server_cert_ssh_known_hosts`, ssh accepts the host key of a node on first contact.
A spoofed node on that first contact gets the management password. Set
`server_cert_ssh_known_hosts` to a file with the host keys of the conductor VIP and every node.
ssh then refuses an unknown host key.

With `server_cert_validate_certs: false`, the role sends the management password to any server
that answers on the conductor address. Use it only for the first run against a factory
certificate. Set `server_cert_ca_path` when your CA is not in the system trust store.

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `server_cert_apply` | `false` | Write to the cluster. |
| `server_cert_conductor` | required | Conductor VIP host name, for ssh and the REST API. |
| `server_cert_username` | required | Management user for ssh and the REST API. |
| `server_cert_password` | required | Password of that user. |
| `server_cert_port` | `4343` | REST API port and web server port on each node. |
| `server_cert_validate_certs` | `true` | Validate the REST API certificate. |
| `server_cert_ca_path` | `""` | CA file for the REST API check. Empty uses the system trust store. |
| `server_cert_ssh_known_hosts` | `""` | known_hosts file for ssh. Empty accepts a new host key. |
| `server_cert_fullchain` | required | Absolute path of the PEM certificate and chain on the control node. |
| `server_cert_privkey` | required | Absolute path of the PEM private key on the control node. |
| `server_cert_name_prefix` | required | Name prefix, letters and digits. Cleanup deletes old unbound names with it. |
| `server_cert_paths` | required | List of `path`, `suffix` and `bind_fields` (`switch-cert`, `captive-portal-cert`). The served certificate check needs `switch-cert`. |
| `server_cert_cleanup` | `true` | Delete old unbound names with the prefix. |
| `server_cert_scp_host` | required | Address the conductor uses for scp. |
| `server_cert_scp_user` | `""` | Empty selects the temp account mode. |
| `server_cert_scp_password` | `""` | Password of the scp user in the remote scp host mode. |
| `server_cert_scp_dir` | `""` | Absolute directory on the scp host. |
| `server_cert_scp_delegate` | `""` | Inventory host that receives the p12. Empty uses `server_cert_scp_host`. |
| `server_cert_temp_user` | `certdrop` | Temp account name. |

`ansible-doc -t role samwiseg0.aruba_cert.server_cert` shows the full descriptions.

## Tested

A private predecessor of this role proved the device workflow on hardware. The cluster was a
Mobility Conductor pair and two Mobility Controllers in one group, on ArubaOS 8.13.1.2 LSR. The
management user had the `root` role. The predecessor sent CLI input through heredocs. This release
sends it with printf on stdin. That input path ran only against a mock. The report mode of this
release ran read-only on ArubaOS 8.13.3.0. The remote scp host mode and the `master` row type of
older AOS 8 releases are untested.

## License

GPL-3.0-or-later. See [LICENSE](LICENSE).
