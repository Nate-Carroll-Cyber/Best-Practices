
### Identity and Access Management (IAM)
The first line of defense is ensuring only authorized entities can request a connection.

* Instead of letting the data directory with a blank password or relying on your OS environment, force a password prompt during creation via the -W (or --pwprompt) flag:
```
initdb -D /var/lib/postgres/data -W
```
  * ** If the database is already running and you missed the initdb -W step, restart and authenticaten as the superuser. Run the following it fix it immediately:
```
ALTER ROLE postgres WITH PASSWORD 'your_strong_password';
```
* Run as a Dedicated User: Never run PostgreSQL as root. Use a dedicated, low-privilege OS user (default postgres).
* Limit the SUPERUSER and CREATEROLE attributes to the absolute minimum number of admins.
* Use least-privilege service accounts for AI agents/tools — grant only necessary SELECT/INSERT on vector tables.
```
CREATE ROLE ai_tool_role WITH LOGIN;
GRANT SELECT, INSERT ON TABLE agent_embeddings TO ai_tool_role;
```
* Disable insecure authentication mechanisms like trust or md5. 
  * ** Avoid peer for remote; prefer certificate-based auth (cert) for service-to-service. 
  * ** Use peer only for local Unix socket connections by the postgres OS user.
  * ** If you don't trust other local users on the system, restrict local connections through filesystem permissions
* Integrate Postgres with 2FA solutions like Duo Security via PAM modules.
* Leverage external authentication: For enterprise environments, use LDAP, Kerberos, or OAuth2 (via extensions like pgsql-ldap or pgjwt) to centralize identity management instead of local Postgres roles.
* Remove unnecessary PUBLIC privileges — Revoke defaults like CREATE on public schema:
```
REVOKE CREATE ON SCHEMA public FROM PUBLIC;.
```
* Enforce RLS + session variables to isolate embeddings / documents per tenant or user — prevents cross-tenant leakage in vector similarity searches.
```
ALTER TABLE agent_embeddings ENABLE ROW LEVEL SECURITY;
```
* Define policies that prevent agents from seeing or mutating each other’s state.
```
CREATE POLICY agent_isolation_policy ON agent_embeddings
USING (agent_id = current_setting('app.agent_id')::uuid);
```
  * ** Agents / apps set session variables (SET app.current_tenant = '…') to enforce isolation.

* ```pg_hba.conf``` (Client Authentication)
This file defines which hosts can connect, which databases they can access, and which authentication method is required.
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# 1. Local 'postgres' superuser access via Unix socket using Peer auth
local   all             postgres                                peer

# 2. Local applications on the same server (Unix socket)
local   all             all                                     scram-sha-256

# 3. Trusted App Server via encrypted tunnel (Internal Network)
# Use 'hostssl' to reject any non-encrypted connection attempts
hostssl all             app_user        10.0.5.0/24             scram-sha-256

# 4. Service-to-service communication using Client Certificates
# Requires sslmode=verify-full on the client
hostssl all             svc_account     192.168.1.50/32         cert

# 5. DENY ALL ELSE (Implicitly, but good to be explicit in mind)
# Do not use 'trust' or 'md5'
```

* ```postgresql.conf``` (Server Settings)
These settings enforce global encryption, hashing standards, and performance guardrails.
```
# --- Connectivity and Encryption ---
listen_addresses = '10.0.5.15'          # Only listen on internal IP
ssl = on                                # Enforce SSL
ssl_ciphers = 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384' 

# --- Authentication ---
password_encryption = scram-sha-256     # Modern salted hashing

# --- Monitoring and Auditing ---
logging_collector = on
log_connections = on                    # Audit successful logins
log_disconnections = on                 # Audit session length
log_statement = 'ddl'                   # Log all schema changes
log_line_prefix = '%m [%p] %u@%d '      # Timestamp, PID, user, and DB
shared_preload_libraries = 'pgaudit'    # Require pgAudit for deep tracking

# --- Resource Protection ---
statement_timeout = '10s'               # Kill runaway agent queries
max_connections = 100                   # Prevent DoS via connection exhaustion
```
* If you are in a "multi-tenant" OS environment (where other users have shell access to the server), restrict the Unix socket itself through filesystem permissions within the ```postgresql.conf``` file:
```
# --- Adjust the socket permissions so only the postgres user and a specific group can even "see" the connection point ---
# Set permissions so only the owner and group can read/write to the socket.
unix_socket_permissions = 0770 

# Optional: Ensure the socket belongs to a specific group.
unix_socket_group = 'postgres'
```


### Network Security & Connection Handling
* By default, some installations listen on all interfaces (*). You should explicitly bind PostgreSQL only to the internal IP address used by your application servers.
```
# postgresql.conf
listen_addresses = '10.0.0.5'
```
* Restrict the server to modern, secure encryption suites.
```
# postgresql.conf
ssl = on
ssl_ciphers = 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384'
```
  * ** Client-Side Verification: On your application connection string, use sslmode=verify-full. This forces the client to verify that the server’s certificate is valid and that the hostname matches, preventing Man-in-the-Middle attacks.
* Use the "Least Privilege" network approach. 
  * ** Instead of broad rules, create specific entries for each application. Do not allow your entire VPC/network if only two servers need access. Only specify the exact database and user for each IP range.
```
# pg_hba.conf 
# TYPE    DATABASE     USER          ADDRESS          METHOD
# Allow only the App Server to reach the Production DB
hostssl   app_db       app_user      10.0.0.15/32     scram-sha-256
```
  * ** OS Firewall: Use iptables or ufw to block any traffic to port 5432 that doesn't originate from your trusted application subnet.
  * ** Cloud Security Groups: If on AWS/Azure, set a Security Group rule that allows inbound traffic on port 5432 only from the Security Group ID of your application tier.
* Deploy a Connection Pooler (PgBouncer). Use PgBouncer to manage connection limits, prevent Denial of Service (DoS) from "runaway" processes, and avoid exposing primary credentials to application code.
  * ** Manage Limits: Set max_connections in Postgres to a safe level and use PgBouncer to queue incoming requests.
  * ** Prevent DoS: Configure statement_timeout in Postgres (e.g., 5s) so that "runaway" queries from AI agents or bugs don't hang all available slots.
  * ** Credential Offloading: Applications connect to PgBouncer; PgBouncer connects to the database. This allows you to rotate database passwords without manually updating every microservice's environment variables.


### Permissions & Internal Authorization
* Revoke Public Defaults: By default, PostgreSQL allows any user to create objects in the public schema. This ensures only the database owner or superusers can create new objects in the public space.
```
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
```
* Implement Row-Level Security (RLS). RLS is the gold standard for multi-tenant applications or AI agents. It acts as a "filter" that is automatically applied to every query, regardless of what the user types.
```
# Enable RLS: You must explicitly turn this on for each sensitive table.
ALTER TABLE sensitive_table ENABLE ROW LEVEL SECURITY;

# Define a Policy: Create a rule that uses a session variable (like app.current_tenant) to filter rows.
CREATE POLICY user_isolation_policy ON sensitive_table 
FOR SELECT 
USING (tenant_id = current_setting('app.current_tenant')::uuid);
```
* Secure "Security Definer" Functions
  * ** Sometimes a low-privilege user needs to perform a high-privilege task (like updating a password). You use SECURITY DEFINER functions for this, but they are vulnerable to "search path hijacking".
  * ** Always set an explicit search_path and use a trusted schema.
```
CREATE FUNCTION change_password(uname TEXT, new_pass TEXT)
RETURNS VOID AS $$
  -- logic here
$$ LANGUAGE plpgsql
   SECURITY DEFINER
   -- Secure the path to prevent hijacking
   SET search_path = trusted_schema, pg_temp;
```
* Role-Based Access Control (RBAC) - Instead of granting permissions to individuals, create roles based on job functions (Principle of Least Privilege).
  * ** Read-Only Roles: For analytics or reporting tools.
  * ** Data Ingestion Roles: For AI agents that only need to INSERT into vector tables.
```
CREATE ROLE ai_agent_role;
GRANT SELECT, INSERT ON TABLE vector_embeddings TO ai_agent_role;
-- Only grant specific columns if possible
GRANT UPDATE (last_processed_at) ON TABLE tasks TO ai_agent_role;
```


### Data Protection (At Rest)
 * Data Integrity (Data Checksums). You must enable data checksums during the initdb phase using the --data-checksums flag.
  * ** This allows PostgreSQL to verify the integrity of each page as it is read from disk, alerting you immediately if the data has changed since it was written.
```
initdb -D /var/lib/postgres/data --data-checksums
```
  * ** AWS EBS: When creating an EBS volume for your EC2 instance or RDS cluster, select the Enable Encryption checkbox and choose a KMS (Key Management Service) key.
  * ** Azure Disk: Use Azure Disk Encryption (ADE) or Encryption at Rest with Platform-Managed Keys, which is enabled by default for most managed PostgreSQL instances.
  * ** Managed Services: In AWS RDS or Azure Database for PostgreSQL, Transparent Data Encryption (TDE) is handled by the platform using the underlying storage encryption mentioned above.
  * ** Filesystem-Level Encryption (LUKS)
```
# Format the Partition: Identify your disk (e.g., /dev/sdb1) and initialize LUKS.
cryptsetup luksFormat /dev/sdb1

# Open the encrypted device and create a filesystem.
cryptsetup open /dev/sdb1 pg_data_encrypted
mkfs.ext4 /dev/mapper/pg_data_encrypted

# Create the directory on the encrypted drive
sudo mkdir -p /mnt/pg_encrypted/data

# Give ownership to the postgres system user
sudo chown -R postgres:postgres /mnt/pg_encrypted/data

# Set strict permissions (700 is standard for Postgres data)
sudo chmod 700 /mnt/pg_encrypted/data

# Stop PostgreSQL
sudo systemctl stop postgresql

# Copy the data: Use rsync to preserve permissions and ownership from the old directory (usually /var/lib/postgresql/14/main/) to the new one.
sudo rsync -av /var/lib/postgresql/14/main/ /mnt/pg_encrypted/data/

# Edit the configuration file and locate the data_directory parameter.
sudo nano /etc/postgresql/14/main/postgresql.conf.
   ...
data_directory = '/mnt/pg_encrypted/data'

# Restart PostgreSQL.
sudo systemctl start postgresql.

```
  * ** Salted Hashing for Passwords (pgcrypto)
```
-- Enable the pgcrypto extension in your database: 
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Generate a salted hash and store a new password with a strong algorithm like Blowfish (bf).
INSERT INTO users (username, password_hash) 
VALUES ('user1', crypt('my_secure_password', gen_salt('bf')));
Verify the Password: Compare the provided plaintext against the stored hash.

-- Returns true if it matches
SELECT (password_hash = crypt('entered_password', password_hash)) AS authorized 
FROM users WHERE username = 'user1';
```
  * ** Application-Level Encryption (Field Level)
```
-- - Instead of storing plaintext, you use functions like pgp_sym_encrypt to store the data and pgp_sym_decrypt to retrieve it using a secret key.
INSERT INTO users (ssn) VALUES (pgp_sym_encrypt('123-45-6789', 'your_secret_key'));
```

### Data Protection (In Transit)
* Enforce scram-sha-256 (the modern, salted password hashing standard) in postgresql.conf (password_encryption = scram-sha-256) and pg_hba.conf (set auth method to scram-sha-256 for all connections).
```
	ssl = on
	password_encryption = scram-sha-256
```	
* Use certificate authentication: For high-trust environments, replace password auth with client certificates (sslmode = verify-full) to ensure only trusted clients can connect.
  * ** Certificate creation
```
# Generate the CA Private Key
openssl genrsa -aes256 -out ca.key 4096

# Create the Self-Signed Root Certificate (ca.crt)
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt

# Use the CA to Sign Your Postgres Server Certificate
openssl req -new -key server.key -out server.csr -subj "/CN=myhost.com"

# openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365

# Move the files in their respective locations as defined in your hardening guidance.
	File,Location,Access Level
	ca.crt,Both Server and Client,Public (Trusted)
	server.crt,Server (/var/lib/postgresql/data/),Public
	server.key,Server (/var/lib/postgresql/data/),Private (chmod 0600)
	ca.key,Secure Offline Storage,Ultra-Secret
```
  * ** Implement ```verify-full``` on Server.
```
# Server Configuration (```postgresql.conf```)
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'ca.crt' # Optional, used if you want to verify clients too
```
  * ** Store the certificate in a standard configuration directory ```/etc/pg-certs/ca.crt``` or ```/app/certs/ca.crt```.
  * ** Set the correct permissions.
```
# Change Ownership: Ensure the app user owns the file.
sudo chown myuser:myuser /etc/pg-certs/ca.crt

# Restrict Access: The certificate should be readable by the user but not writable by others.
chmod 644 /etc/pg-certs/ca.crt
```
  * ** When using the connection string, always use the full absolute path.
```
# Checks that the server certificate is signed by a trusted CA and that the server's hostname matches the name on the certificate.
postgres://user:pass@myhost.com:5432/db?sslmode=verify-full&sslrootcert=/etc/pg-certs/ca.crt
```
  * ** Verify active encryption.
```
-- Run this query to check the SSL status, version, and cipher suite for all active backends:
SELECT datname, usename, ssl, version, cipher, bits, client_dn 
FROM pg_stat_ssl 
JOIN pg_stat_activity ON pg_stat_ssl.pid = pg_stat_activity.pid;
```


### Auditing & Monitoring
Security is a continuous process of observation.

pgAudit Extension: Install and configure pgAudit for detailed session and object-level logging (e.g., logging every UPDATE on a specific financial table).

* Configure ```postgresql.conf``` for detailed logging.
```
# --- Connection Auditing ---
log_connections = on                # Logs every successful connection
log_disconnections = on             # Logs the end of every session
log_failed_connections = on         # Captures failed login attempts for brute-force detection

# --- Statement Logging ---
# 'ddl' logs all structural changes; use 'mod' or 'all' for higher security
log_statement = 'ddl'               
log_min_duration_statement = 250    # Logs queries exceeding 250ms

# --- Log Formatting ---
# %m: timestamp, %p: process ID, %q: stop if not backend, %u: user, @%d: database
log_line_prefix = '%m [%p] %q%u@%d '
```

SIEM Integration: Forward logs to a central analyzer (Splunk, ELK, or Datadog) to alert on unusual spikes in failed logins or data exports.


### Monitoring & Auditing

* Monitor SSL Usage: Use the pg_stat_ssl view to verify SSL connections
```
SELECT * FROM pg_stat_ssl;
```




Check SSL Status: Use PQsslInUse() or query pg_stat_ssl to confirm encryption is active


Configure ```postgresql.conf``` to log:
```
	log_connections = on
	log_disconnections = on
	log_line_prefix = '%m [%p] %q%u@%d '
	log_statement = 'ddl'          # or 'mod' / 'all' in high-security
	log_min_duration_statement = 250
```
Failed login attempts (log_failed_connections = on).

Connection/disconnection events (log_connections = on, log_disconnections = on).

DDL changes (log_statement = 'ddl'). Set log_statement = 'ddl' (or 'all' for debug) in postgresql.conf to log schema changes.

Slow queries (log_min_duration_statement = 500 to log queries taking over 500ms).

Use pgAudit for granular auditing: Install the pgAudit extension to track specific actions (e.g., table access, role changes) with detailed logs for compliance (e.g., GDPR, HIPAA).

Use pgAudit extension — For detailed statement-level auditing (SELECT/INSERT/UPDATE per object).

Install pgAudit to log detailed DML/DDL activities (e.g., CREATE TABLE, UPDATE queries).

Configure pgAudit.log to capture specific events (e.g., WRITE, ROLE).

Deploy tools like pgBadger or commercial DAM solutions to detect anomalies (e.g., brute-force attacks).

Monitor for anomalies: Use tools like Prometheus + Grafana, Datadog, or New Relic to track:
Unusual login patterns (e.g., multiple failed attempts from unknown IPs).

Monitor log_connections and log_disconnections to track access patterns.

Sudden spikes in data export volume.

Unauthorized role or permission changes.

Forward logs to a SIEM tool (e.g., Splunk, ELK) for real-time analysis and alerting.

Set up alerts for critical events (e.g., superuser login outside business hours).




8. Permission Management
Principle of Least Privilege: Grant minimal necessary permissions

Follow least privilege principle: Create dedicated roles for each application, service, or user with only the permissions they need (e.g., a read-only role for analytics, a write-only role for data ingestion). Avoid using the postgres superuser for application connections.

Restrict superuser access: Limit SUPERUSER and CREATEROLE privileges to a small number of trusted administrators. Use NOLOGIN roles for group permissions to prevent direct login.

GRANT SELECT, UPDATE (specific_columns) ON table TO role;

Role-Based Access: Create specific roles with limited permissions rather than using superuser accounts



9. Firewall & Network
Configure firewalls to restrict database access to authorized IPs/applications

Don't expose PostgreSQL directly to untrusted networks

These practices work together to create a comprehensive security posture for your PostgreSQL deployment.

Restrict network access: In postgresql.conf, set listen_addresses to only the IP addresses that need to connect (avoid * in production). Use firewall rules (cloud security groups, on-prem firewalls) to block unauthorized network traffic to Postgres (default port 5432).

Configure pg_hba.conf to allow connections only from trusted IPs/subnets.

	hostssl appdb app_user 10.0.0.0/24 scram-sha-256


Use a firewall (e.g., iptables, ufw) to block unauthorized access to Postgres ports (default: 5432).

Limit concurrent connections: Set max_connections to a reasonable value and use connection limits per role (ALTER ROLE <role> CONNECTION LIMIT <number>) to prevent denial-of-service (DoS) attacks.

Set query timeout limits to protect against runaway agent loops.

	statement_timeout = '5s'

Use connection poolers: Deploy tools like PgBouncer to proxy connections, reduce direct database exposure, and enforce rate limiting.

Restrict Unix domain sockets — Set strict filesystem permissions (e.g., unix_socket_permissions = 0770) and group ownership.

Consider connection pooling (PgBouncer) with authentication offload to avoid credential exposure in agent code.




10. Patching & Updates

Keep Postgres up to date: Regularly apply security patches and minor version updates to address vulnerabilities. Use official packages from the PostgreSQL Global Development Group (PGDG) or managed cloud services that automate updates.

Update extensions: Vulnerabilities often exist in third-party extensions (e.g., pgvector, pgcrypto). Only use trusted extensions from official sources, and update them alongside Postgres.

Remove unused extensions: Disable or uninstall extensions that are no longer needed to reduce the attack surface.

**### Maintenance & Integrity
Patch Management: PostgreSQL minor versions include critical security fixes. Apply these quarterly or as released.

Extension Management: Regularly audit pg_extension. Remove any unused extensions (like uuid-ossp if you use native gen_random_uuid()) to reduce the attack surface.

Statement Timeouts: Prevent resource exhaustion by setting a statement_timeout (e.g., 5s) to kill runaway queries before they crash the server.


**
