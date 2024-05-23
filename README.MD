# PGT-Proxy (Docker)

PostgreSQL TLS Proxy (PGT-Proxy) is an intermediary server to easily and securely connect TLS enabled PG clients 
to TLS enabled PG servers. This repository contains the Docker deployment for PGT-Proxy. The source code for PGT-Proxy 
lives separately in another repository.

- Pushed to DockerHub [repository](https://hub.docker.com/r/ambarltd/pgt-proxy)
- Rust source code available [on GitHub](https://github.com/ambarltd/pgt-proxy)

## Usage

**Step 1.** Create a Dockerfile based on pgt-proxy.

```Dockerfile
FROM ambarltd/pgt-proxy:latest

# PGT-Proxy will need to trust a set of certificate authorities 
# (to be specified at runtime through "--client-ca-roots-path")
# 
# Use a prepackaged set of root authorities such as RDS's CAs or Firefox's default trusted CAs, living at these directories:
#   /etc/pgt_proxy/client_tls/aws_rds/
#   /etc/pgt_proxy/client_tls/firefox/
# Or specify trusted CAs in the directory /etc/pgt_proxy/client_tls/custom_cas/ (e.g., Google Cloud SQL uses a custom CA)
RUN COPY path/to/postgres_destination_server/certificate_authority_certificates_in_pem_format/ /etc/pgt_proxy/client_tls/custom_cas/

# PGT-Proxy will need to serve TLS traffic, which requires a certificate and its corresponding private key
# (to be specified at runtime through "--server-private-key-path", "--server-certificate-path")
# 
# The certificate needs to be trusted by all pg clients connecting through PGT-Proxy. 
# Thus it is recommended to use a genuine TLS certificate issued by a public certificate authority
# that is already trusted by default by pg clients.
RUN COPY path/to/pgt_proxy/tls_certificate.pem /etc/pgt_proxy/server_tls/certificate.pem
RUN COPY path/to/pgt_proxy/tls_private_key.pem /etc/pgt_proxy/server_tls/key.pem

# Set the arguments passed to PGT-Proxy.
CMD ["--server-private-key-path", "/etc/pgt_proxy/server_tls/key.pem", \
    "--server-certificate-path", "/etc/pgt_proxy/server_tls/certificate.pem", \
    "--server-port", "5432", \
    "--client-connection-host-or-ip", "destination.host.example.amazonaws.com", \
    "--client-connection-port", "5432", \
    "--client-tls-validation-host", "destination.host.example.amazonaws.com", \
    "--client-ca-roots-path", "/etc/pgt_proxy/client_tls/custom_cas/"]
```

**Step 2.** Deploy your Dockerfile to a machine that allows inbound and outbound traffic on the `--server-port` 
(e.g., 5432).

**Step 3.** Using DNS, point the hostname inside `/etc/pgt_proxy/server_tls/certificate.pem` to the IP address 
of the machine in step 2.

**Step 4.** Connect to PGT-Proxy via the machine in step 2, using your favorite pg client. E.g., 

```bash
psql 'sslmode=verify-full host=pgtproxy.example.com port=5432 user=admin password=pass dbname=postgres channel_binding=disable'
```

**Suggestions**
- Deploy PGT-Proxy on AWS EC2, AWS Fargate, GCP CloudRun, Kubernetes, or wherever you feel most comfortable.
- Use infrastructure as code where possible (e.g., Terraform, Pulumi).
- Integrate deployment and TLS certificate issuance/renewal into your CI/CD system.
- For high availability in vendors such as AWS and GCP, deploy PGT-Proxy redundantly in more than one availability zone.
- Restrict network traffic to the machine that runs PGT-Proxy. E.g., only allow traffic from trusted IPs.
