# samwiseg0.aruba_cert.server_cert

This role pushes a TLS server certificate to an Aruba Mobility Conductor cluster. The role only
reports unless `server_cert_apply` is true and the run is not in check mode.

Read the [collection README](../../README.md) for install, a quick start and the scp modes.
This command lists every variable.

```bash
ansible-doc -t role samwiseg0.aruba_cert.server_cert
```
