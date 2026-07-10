# wstunnel mTLS material

Place the wstunnel **Certificate Authority (CA)** certificate here as
`tunnel-ca.crt`.

A CA is just a trust anchor you generate yourself: one key pair whose certificate
"signs" the home client's certificate. Traefik is told to trust this CA, so it
accepts any client cert the CA signed (here, only the home box's) and rejects
everything else. Nothing is bought or registered — it's entirely self-issued.

Traefik mounts this dir read-only at `/etc/traefik/mtls` and uses `tunnel-ca.crt`
to verify the home client's certificate on the `wstunnel.DOMAIN` router
(`RequireAndVerifyClientCert`). **Only the public CA cert belongs here** — the
CA's signing key (`tunnel-ca.key`) and the client cert+key must **not** live on
the VPS.

## Generate the material once

Run on a trusted machine and keep `tunnel-ca.key` offline:

```bash
# CA
openssl genrsa -out tunnel-ca.key 4096
openssl req -x509 -new -nodes -key tunnel-ca.key -sha256 -days 3650 -subj "/CN=wstunnel-ca" -out tunnel-ca.crt

# Home client cert, signed by the CA
openssl genrsa -out home-client.key 4096
openssl req -new -key home-client.key -subj "/CN=home" -out home-client.csr
openssl x509 -req -in home-client.csr -CA tunnel-ca.crt -CAkey tunnel-ca.key -CAcreateserial -days 825 -sha256 -out home-client.crt
```

## Distribute

- Copy **only** `tunnel-ca.crt` → this directory (`traefik-data/mtls/`) on the VPS.
- Copy `home-client.crt` + `home-client.key` → the home box (for the wstunnel
  client).

This directory is git-ignored (see `.gitignore`), so nothing here is committed.
