# SSL/TLS Certificate Management

This document explains the SSL/TLS certificate strategy for the proxy architecture, including why we use Let's Encrypt with DNS-based validation and how the certificate workflow integrates with HAProxy.

## Why Let's Encrypt?

Let's Encrypt provides free, automated SSL/TLS certificates with a few key advantages:

**Automation-Friendly**: Designed from the ground up for programmatic certificate issuance and renewal
**Trusted**: Certificates are trusted by all major browsers and operating systems
**Short-Lived**: 90-day certificate lifetime encourages automation and reduces the impact of key compromise
**No Cost**: Removes the financial barrier to encrypting everything

For a homelab proxy handling multiple AI nodes, Let's Encrypt means we can issue certificates for each hostname without licensing costs.

## The DNS-01 Challenge

Let's Encrypt needs to verify you control the domain before issuing certificates. There are two common validation methods:

**HTTP-01**: Let's Encrypt makes an HTTP request to your domain at a specific path
**DNS-01**: Let's Encrypt checks for a specific TXT record in your DNS zone

### Why We Use DNS-01

The proxy nodes serve internal resources that we don't want to expose to the public internet for validation. Additionally, we're requesting certificates for private hostnames that may not be reachable from Let's Encrypt's validation servers.

DNS-01 validation works regardless of whether your hosts are publicly accessible. As long as you can create TXT records in your public DNS zone (Route53 in this case), Let's Encrypt can validate your ownership.

This allows us to get valid SSL certificates for `hal9000-api.kubernerdes.com` even if that hostname isn't publicly routable during the validation process.

## Certificate Workflow

### Step 1: Request Certificates

Certbot (the Let's Encrypt client) requests certificates for all your API hostnames in a single operation:

```bash
# Example: Request certificates for all AI node API endpoints
sudo certbot certonly --dns-route53 \
  -d hal9000-api.kubernerdes.com \
  -d brainiac-api.kubernerdes.com \
  -d kitt-api.kubernerdes.com \
  --non-interactive \
  --agree-tos \
  --email your-email@example.com
```

**What happens here:**
1. Certbot contacts Let's Encrypt and requests certificates for the three hostnames
2. Let's Encrypt responds with DNS challenges (specific TXT record values to create)
3. Certbot uses AWS credentials to create TXT records in your Route53 hosted zone
4. Let's Encrypt queries DNS, verifies the TXT records exist
5. Upon successful validation, Let's Encrypt issues the certificates
6. Certbot downloads and stores them in `/etc/letsencrypt/`

The `--dns-route53` plugin handles all the DNS record creation and cleanup automatically.

### Step 2: Convert to HAProxy Format

Certbot stores certificates in its own directory structure with separate files for the certificate chain and private key. HAProxy expects a single PEM file containing both.

For each hostname, concatenate the certificate and key:

```bash
# Example for hal9000
sudo bash -c 'cat /etc/letsencrypt/live/hal9000-api.kubernerdes.com/fullchain.pem \
    /etc/letsencrypt/live/hal9000-api.kubernerdes.com/privkey.pem \
    > /etc/haproxy/certs/hal9000-api.kubernerdes.com.pem'

# Example for brainiac
sudo bash -c 'cat /etc/letsencrypt/live/brainiac-api.kubernerdes.com/fullchain.pem \
    /etc/letsencrypt/live/brainiac-api.kubernerdes.com/privkey.pem \
    > /etc/haproxy/certs/brainiac-api.kubernerdes.com.pem'

# Example for kitt
sudo bash -c 'cat /etc/letsencrypt/live/kitt-api.kubernerdes.com/fullchain.pem \
    /etc/letsencrypt/live/kitt-api.kubernerdes.com/privkey.pem \
    > /etc/haproxy/certs/kitt-api.kubernerdes.com.pem'
```

**Why this format?**
HAProxy reads the certificate file once at startup. Having everything in a single file simplifies the configuration - you just point HAProxy at a directory of PEM files, and it automatically maps certificates to hostnames based on the filename.

### Step 3: Reload HAProxy

After updating certificates, HAProxy needs to reload its configuration:

```bash
sudo systemctl reload haproxy
```

HAProxy supports graceful reloads - existing connections continue uninterrupted while new connections use the updated certificates.

## Certificate Renewal

Let's Encrypt certificates expire after 90 days. Certbot can handle renewals automatically:

```bash
# Check what would be renewed (dry run)
sudo certbot renew --dry-run

# Actually renew certificates nearing expiration
sudo certbot renew
```

Typically, you'd run this via cron or systemd timer. After renewal, you need to re-concatenate the certificates and reload HAProxy. A post-renewal hook can automate this:

```bash
# Example renewal hook script (conceptual)
#!/bin/bash
# Regenerate HAProxy certificate bundles
cat /etc/letsencrypt/live/hal9000-api.kubernerdes.com/fullchain.pem \
    /etc/letsencrypt/live/hal9000-api.kubernerdes.com/privkey.pem \
    > /etc/haproxy/certs/hal9000-api.kubernerdes.com.pem

# Repeat for other hostnames...

# Reload HAProxy
systemctl reload haproxy
```

## AWS Route53 Credentials

The `--dns-route53` plugin requires AWS credentials with permissions to modify DNS records. Certbot typically looks for credentials in:

- Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
- AWS credentials file (`~/.aws/credentials`)
- IAM instance role (if running on EC2)

**Minimum required permissions:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:GetChange"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "route53:ChangeResourceRecordSets",
      "Resource": "arn:aws:route53:::hostedzone/YOUR_ZONE_ID"
    }
  ]
}
```

## Security Considerations

**Private Key Protection**: The concatenated PEM files contain private keys. Ensure proper file permissions:
```bash
sudo chmod 600 /etc/haproxy/certs/*.pem
sudo chown root:root /etc/haproxy/certs/*.pem
```

**AWS Credential Scope**: Use IAM credentials with the minimum required Route53 permissions, scoped to only the hosted zone you need to modify.

**Certificate Monitoring**: Monitor certificate expiration dates. While automation should handle renewals, having alerting on certificates approaching expiration provides a safety net.
