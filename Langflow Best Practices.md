## Langflow Hardening Guidance

Securing Langflow requires moving beyond "out-of-the-box" settings, which are often optimized for local development rather than production security. Platforms like Langflow, n8n, and other workflow automation tools are increasingly targeted because they often run with broad API access and are deployed outside of standard security review processes.

### Network Architecture & Segmentation
Langflow is a code-execution platform; if it is compromised, the attacker essentially has a shell on the underlying host.

* The "Private-First" Rule: Deploy Langflow within a Private VPC. Langflow neither enforces isolation between users within a single Langflow process, nor restricts access to the local disk or network resources. In the event that Langflow could execute untrusted or LLM-generated code, consider using isolated or containerized execution environments. Access should only be possible via VPN or ZTNA (Zero Trust Network Access). 
* Secure API Gateway: Use a reverse proxy (Nginx, Traefik) or a dedicated API gateway to handle TLS termination, rate limiting, and centralized authentication.
* SSRF Protection: Enable and tune SSRF (Server-Side Request Forgery) protections on components capable of making API requests. This prevents the agent from being used to probe internal loopback ranges or cloud metadata endpoints.
* Sandboxed Execution: Langflow utilizes Python's exec(). For high-security environments, run the Langflow worker in a sandbox like gVisor or a micro-VM (Firecracker) to isolate the kernel from the AI's code execution.
* Disable Public Flow Building: If your team does not require public flow features, disable them entirely in the configuration to reduce the attack surface.
* Strict Egress Filtering: AI agents are high-risk for data exfiltration. Use a firewall to restrict outbound traffic to an "allowlist" of verified API endpoints (e.g., api.openai.com).

### Authentication & Identity Management
Langflow’s JWT docs say RS256 provides better security for production by using asymmetric keys, and they also document RS512. They specifically recommend Docker or Kubernetes with RS256 for production deployments that need enhanced security.

* Audit environment variables and secrets** on any publicly exposed Langflow instance. Rotate API keys, database passwords, and cloud credentials as a precaution.
* JWT Hardening: Avoid the default symmetric HS256 if possible. Use RS256 (Asymmetric) or RS512 for production to prevent token forgery if the server secret is leaked.
  * ** Set ```LANGFLOW_ALGORITHM="RS256"``` in your .env.
* Store private keys outside the image.
* Rotate keys on a defined schedule.
* Manually generate keys: ```openssl genrsa -out private_key.pem 2048```.
* OAuth2/OIDC: Integrate with corporate IDPs (Okta, Azure AD) rather than relying on local database users.
* Non-Root Execution: In Docker, ensure the process runs as a low-privilege user (UID 1000) rather than root to prevent container escape.

### Secret Management & Environment Variables
* Externalize Secrets: Never hard-code keys in the Langflow UI. Use ```LANGFLOW_CONFIG_DIR``` and inject secrets via environment variables or a sidecar secret manager (HashiCorp Vault).
* Database Encryption: Ensure the disk or volume hosting the Langflow SQLite/PostgreSQL database is encrypted at rest (AES-256). In the default deployment configuration of Langflow, which uses a local SQLite database, the attacker can access this data without needing any credentials.
```
# find / -iname langflow.db
# cd /app/.venv/lib/python3.12/site-packages/langflow
# sqlite3 langflow.db `select * from user;`
# sqlite3 langflow.db `select id, name, user_id from where user_id !="";`
# sqlite3 langflow.db `select * from variable;`
```
* If Langflow is configured to use a different database backend (e.g., PostgreSQL), the attacker may still be able to retrieve the necessary credentials by inspecting environment variables.
* Read-only root filesystem where possible.
  * ** ```readOnlyRootFilesystem: true``` for runtime containers.

### Fix CORS before going live
Langflow’s docs specifically warn that the default CORS settings can be a security risk in production, because any website may be able to make requests to your Langflow API and include credentials. 

* Never leave permissive CORS in production.
  * ** Restrict ```LANGFLOW_CORS_ORIGINS``` to specific, known frontend origins only.
  * ** Keep methods and headers narrow.

### Web Headers (Caddy)

```
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

* Caddy already sets X-Forwarded-For, X-Forwarded-Proto, and X-Forwarded-Host by default in reverse_proxy, and it ignores spoofed incoming values for those headers unless you configure trusted proxies.
* ```X-Frame-Options "DENY"``` prevents clickjacking, but will prevent you from embedding a Langflow chat widget into other sites later. If this is purely for internal use within the Langflow UI, ```DENY``` is perfect.
* While Caddy handles WebSockets automatically, it is highly sensitive to timeouts. If a user is idling in the flow builder, Caddy might kill the connection.
  * ** ```flush_interval -1``` is more of a targeted optimization than a baseline hardening setting. 
  * ** ```interest-cohort=()``` is legacy and does not hurt, but it does not buy you much today.
* A Content Security Policy (CSP) may break Langflow unless validated in-browser.
  * ** ```connect-src 'self' wss:``` only allows same-origin HTTP(S) plus WebSockets, and may need adjustment if the frontend ever talks to anything else directly from the browser. Test the full UI, then tighten it iteratively.
```
default-src 'self';
script-src 'self' 'unsafe-inline';
style-src 'self' 'unsafe-inline';
img-src 'self' data:;
connect-src 'self' wss:;
```
  * ** A hardened Caddyfile for reference:
```
{$DOMAIN} {
    encode gzip zstd

    # Security Headers
    header {
        # HSTS (1 year)
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        # Prevent MIME-sniffing
        X-Content-Type-Options "nosniff"
        # Prevent Clickjacking (Set to SAMEORIGIN if you use widgets on the same domain)
        X-Frame-Options "DENY"
        # Privacy-conscious referrer info
        Referrer-Policy "strict-origin-when-cross-origin"
        # Disable unused hardware/APIs
        Permissions-Policy "geolocation=(), microphone=(), camera=(), interest-cohort=()"
        # Basic CSP (Adjust if you use external fonts/scripts)
        Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self' wss:;"
    }

    # Proxy to Langflow
    reverse_proxy langflow:7860 {
        # Standard proxy headers
        header_up X-Forwarded-Proto {scheme}
        header_up X-Forwarded-Host {host}
        
        # Disable response buffering for better streaming/WebSocket performance
        flush_interval -1
    }

    tls {$EMAIL}
}
```

### Patch Information
The vulnerability has been fixed in **Langflow version 1.9.0**. The fix removes the `data` parameter from the `build_public_tmp` endpoint entirely, preventing attackers from supplying malicious flow data. Organizations should upgrade to the latest version immediately.

* Patch aggressively.
* Monitor [Langflow Security](https://github.com/langflow-ai/langflow/blob/main/SECURITY.md) reporting.
* Workarounds:
  * ** Implement network-level access controls** to restrict access to Langflow instances to trusted IP addresses only.
  * ** Deploy a web application firewall (WAF)** to block requests containing suspicious Python code patterns in POST bodies.
  * ** Disable public flow functionality** if not required for business operations.
  * ** Place Langflow instances behind an authentication proxy** to require authentication for all endpoints.

```bash
# Example: Block access to vulnerable endpoint using nginx
location ~ ^/api/v1/build_public_tmp/.*/flow$ {
    deny all;
    return 403;
}

# Example: Restrict Langflow access to internal networks using iptables
iptables -A INPUT -p tcp --dport 7860 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 7860 -j DROP
```

### Detection Strategies
* Monitor web application logs** for POST requests to `build_public_tmp` endpoints containing suspicious `data` payloads.
* Implement network intrusion detection rules** to identify malicious Python code patterns in HTTP request bodies.
* Deploy endpoint detection and response (EDR)** solutions to detect anomalous process execution from Langflow processes.
* Review application-level logging** for flow builds with externally supplied data parameters.
* Enable full API logging to capture the content of flow parameters.

### Indicators of Compromise
* **Unexpected POST requests** to `/api/v1/build_public_tmp/*/flow` endpoints containing `data` parameters with Python code.
* **Unusual process spawning** from the Langflow server process (e.g., shell commands, reverse shells).
* **Network connections** from Langflow servers to unknown external IP addresses.
* **New user accounts or SSH keys** created on systems running Langflow.

---
---
