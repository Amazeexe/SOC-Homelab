# HashiCorp Vault PKI

[← Back to main README](../README.MD) · [Documentation index](README.md)

Vault provides an internal certificate authority for the lab so internal services can use a consistent trust model instead of relying on unmanaged self-signed leaf certificates.

## CA Hierarchy

```text
Homelab-Root-CA              RSA-4096 / 10 years
    |
    +-- Homelab-Intermediate-CA   RSA-4096 / 5 years
            |
            +-- Service leaf certificates   1 year
```

The root and intermediate CAs are separated into distinct Vault PKI mounts, with service certificates issued through the intermediate.

## Root CA

![Vault root CA](images/vault/root-ca.webp)

## Intermediate CA

![Vault intermediate CA](images/vault/intermediate-ca.webp)

## Issued Service Certificates

![Vault service certificates](images/vault/all-service-certificates.webp)

![Vault leaf certificate](images/vault/vault-leaf-certificate.webp)

![ELK leaf certificate](images/vault/elk-leaf-certificate.webp)

## Browser Trust Validation

The Vault-issued Kibana certificate chains through the internal intermediate and root CA trusted by the workstation.

![Kibana trusted Vault certificate](images/vault/kibana-trusted-certificate.webp)

This provides visible proof that the browser is validating the internal PKI rather than being configured to ignore certificate warnings.

## Elastic Trust Chain

The PKI is also part of the telemetry path:

- OPNsense Filebeat validates Logstash.
- Logstash validates Elasticsearch using the internal CA and full server identity verification.
- Elasticsearch presents its HTTP leaf certificate together with the intermediate certificate so clients can build the chain correctly.
- Kibana and Fleet use the internal CA/fingerprint model for trusted Elasticsearch connections.

A certificate-chain failure previously interrupted Elastic Agent publishing even though the services themselves were otherwise running. Correcting the presented Elasticsearch HTTP chain and client trust restored ingestion and reinforced that PKI reliability is part of monitoring reliability.

See [Elastic SIEM & Endpoint Telemetry](elastic-and-endpoint-telemetry.md) and [Lessons Learned](lessons-learned.md) for that troubleshooting path.

## Services Using Internal PKI

The Vault-issued trust model is used across internal services including:

- Elasticsearch / Kibana,
- Logstash ingestion,
- OPNsense management,
- Proxmox,
- OpenVAS,
- Vault itself.

## Secret Handling

Private keys, Vault unseal shares, root tokens, service tokens, passwords, and other authentication material are intentionally excluded from the repository.

Only sanitized configuration and public certificate/trust evidence should be committed.
