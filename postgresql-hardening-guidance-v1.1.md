# PostgreSQL Hardening Guidance v1.1

## Executive Summary

PostgreSQL security depends on coordinated controls across identity, authorization, network access, TLS, operating-system isolation, data protection, query governance, logging, backup, patching, and recovery. No single setting—password authentication, TLS, row-level security, encryption at rest, or a connection pooler—provides a complete security boundary.

For AI agents and automation platforms, assume that queries may be malformed, adversarially influenced, excessively expensive, or broader than the initiating user intended. The database must independently enforce tenant isolation, least privilege, resource limits, and audit requirements.

The core production principles are:

1. Keep PostgreSQL on private networks and permit only explicitly approved clients.
2. Use a dedicated operating-system identity and never run PostgreSQL as root.
3. Give every workload a distinct database login and a narrow, non-login permission role.
4. Prefer short-lived or centrally rotated credentials and strong channel binding.
5. Verify server identity with TLS; do not merely encrypt without verification.
6. Enforce tenant isolation in the database and prevent clients from choosing their own tenant context.
7. Separate application, migration, administration, monitoring, and backup privileges.
8. Protect secrets and encryption keys outside SQL text, logs, shell history, and connection URLs.
9. Apply resource limits per workload and validate connection-pooler behavior.
10. Patch, back up, test recovery, and monitor continuously.

This document is version-aware implementation guidance. Validate parameter names, supported authentication methods, TLS controls, extension compatibility, and managed-service behavior against the exact PostgreSQL version and platform before deployment.

---

## 1. Scope and Deployment Inventory

This guidance applies to self-managed PostgreSQL, containerized PostgreSQL, and managed database services. Some operating-system, filesystem, TLS, extension, and logging controls may be implemented or restricted by the service provider.

Maintain a deployment inventory containing:

- PostgreSQL major and minor version
- Distribution, image, or managed-service engine version
- Host operating system or container image digest
- Extensions and versions, including vector extensions
- Databases, schemas, owners, and application roles
- Network listeners, permitted clients, and connection poolers
- Authentication methods and identity providers
- TLS certificate authority, certificate owner, and expiration
- Backup locations, encryption keys, and recovery objectives
- Data classification and tenant model
- Patch, monitoring, and incident-response owners

Record the authoritative documentation and validation date for every version-sensitive control.

---

## 2. Secure Initialization and Administrative Access

### 2.1 Initialize with explicit authentication policy

Do not depend on packaging defaults. Define local and host authentication deliberately during cluster initialization where the deployment method supports it.

Illustrative self-managed initialization:

```bash
initdb \
  --pgdata=/var/lib/postgresql/data \
  --auth-local=peer \
  --auth-host=scram-sha-256 \
  --pwprompt \
  --data-checksums
```

Before running this command:

- Confirm the path is an empty, intended data directory.
- Run it as the dedicated PostgreSQL operating-system user.
- Verify directory ownership and permissions.
- Confirm that the chosen local authentication policy matches the host trust model.
- Verify that data checksums and authentication options are supported by the deployed version.

Initialization flags do not replace `pg_hba.conf`, network controls, or post-installation role review.

### 2.2 Set and rotate administrative passwords safely

Do not place passwords in SQL files, command arguments, tickets, shell history, or documentation examples. For interactive recovery, use a client facility that prompts securely, such as the `psql` password-changing command:

```text
\password postgres
```

A password change normally does not require restarting PostgreSQL. After changing a credential:

- Update dependent secret stores through a controlled process.
- Test new connections before revoking old access where rotation overlap is required.
- Terminate or revoke sessions when the incident or policy requires it.
- Record the rotation without recording the secret.

Prefer named administrative identities over routine use of the built-in `postgres` role. Reserve emergency access for controlled, audited recovery.

### 2.3 Operating-system identity

- Run PostgreSQL as a dedicated, non-root operating-system user.
- Restrict access to the data, configuration, certificate, log, and socket directories.
- Do not grant interactive shell access to application identities.
- Prevent unrelated users and workloads from reading PostgreSQL process information, environment, backups, or files.
- Apply systemd, SELinux, AppArmor, container, or equivalent controls appropriate to the platform.

---

## 3. Role and Privilege Architecture

### 3.1 Separate login and permission roles

Use `NOLOGIN` roles as permission groups and distinct `LOGIN` roles for workloads and people.

```sql
CREATE ROLE ai_reader NOLOGIN;

GRANT CONNECT ON DATABASE app_db TO ai_reader;
GRANT USAGE ON SCHEMA app TO ai_reader;
GRANT SELECT ON TABLE app.agent_embeddings TO ai_reader;

CREATE ROLE agent_runtime LOGIN;
GRANT ai_reader TO agent_runtime;
```

Set the login credential through the approved secret-management or identity process rather than embedding it in the script.

Separate roles for:

- Read-only application access
- Data ingestion
- Bounded updates
- Schema migration
- Backup and restore
- Monitoring
- Security audit
- Human administration

Applications must not connect as database owners, superusers, migration roles, or roles with `CREATEROLE`, `CREATEDB`, `REPLICATION`, or `BYPASSRLS` unless the function explicitly requires it and the risk is approved.

### 3.2 Schema privileges

Review privileges on every schema. Where compatible with the application and PostgreSQL version:

```sql
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
```

Also review:

- Database `CONNECT`, `CREATE`, and `TEMP` privileges
- Schema `USAGE` and `CREATE`
- Table, sequence, function, procedure, and type privileges
- Default privileges for future objects
- Ownership of databases, schemas, and objects

Example default privileges for objects subsequently created by a designated owner:

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app
  REVOKE ALL ON TABLES FROM PUBLIC;

ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app
  GRANT SELECT ON TABLES TO ai_reader;
```

Default privileges are scoped to the object-creating role and schema; configure them for each relevant owner.

### 3.3 Column and sequence privileges

Grant only required columns where practical:

```sql
GRANT UPDATE (last_processed_at) ON TABLE app.tasks TO agent_writer;
```

Inserting into tables backed by sequences may require separate sequence privileges. Avoid broad schema-wide grants unless justified and reviewed.

### 3.4 Role limits

Apply workload-specific connection and execution limits:

```sql
ALTER ROLE agent_runtime CONNECTION LIMIT 20;
ALTER ROLE agent_runtime SET statement_timeout = '5s';
ALTER ROLE agent_runtime SET lock_timeout = '2s';
ALTER ROLE agent_runtime SET idle_in_transaction_session_timeout = '30s';
```

Timeout values are workload-dependent. Test them against normal queries, migrations, maintenance, and recovery operations. Prefer per-role or per-database limits over a single global timeout that can break administration.

---

## 4. Authentication and `pg_hba.conf`

### 4.1 Authentication policy

- Prohibit `trust` outside narrowly controlled disposable development environments.
- Do not create new MD5 password verifiers; migrate password authentication to SCRAM where supported.
- Use peer authentication only for explicitly trusted local operating-system identities.
- Use certificate or centralized identity integration where operationally appropriate.
- Keep authentication rules narrow by database, role, address, and method.
- Review `pg_hba.conf` from top to bottom because the first matching rule applies.

### 4.2 Example baseline

The following is illustrative and must be adapted to the exact network, roles, databases, certificate policy, and PostgreSQL version:

```conf
# TYPE     DATABASE   USER            ADDRESS          METHOD

# Local emergency administration through the postgres OS identity.
local      all        postgres                         peer

# Application database from one approved application address.
hostssl    app_db     agent_runtime   10.0.5.15/32     scram-sha-256

# Certificate-authenticated service identity. Configure identity mapping
# and client-certificate verification according to the deployed version.
hostssl    app_db     svc_runtime     10.0.5.20/32     cert

# Explicit terminal rejects improve readability and logging expectations.
host       all        all             0.0.0.0/0        reject
host       all        all             ::/0             reject
```

Test the complete rule set before reload. An unmatched connection is rejected even without terminal rules, but explicit rules can make policy intent clearer.

Reload configuration through the approved service-management mechanism and test both expected allows and expected denials.

### 4.3 Certificate authentication

Certificate authentication requires more than `hostssl`:

- A trusted server certificate and private key
- A client that validates the server certificate and hostname
- A trusted client certificate and private key when mutual TLS is required
- Server-side client-certificate verification
- A defined mapping from certificate identity to database role
- Revocation, expiration monitoring, and rotation procedures

Client `sslmode=verify-full` verifies the server certificate and hostname. It does not, by itself, configure or prove client-certificate authentication.

### 4.4 Centralized identity

Use only authentication mechanisms and integrations supported and maintained for the exact platform. LDAP, Kerberos/GSSAPI, PAM, cloud identity, and proxy-based identity each have different trust and availability characteristics. OAuth is not a generic drop-in replacement through arbitrary extensions.

Before adopting an identity extension or proxy, evaluate:

- Maintainer and release history
- Authentication and authorization boundary
- Token audience and expiration validation
- Fail-open versus fail-closed behavior
- Connection-pooling compatibility
- Emergency access
- Auditability and revocation latency

---

## 5. Network Security

### 5.1 Private listeners

Listen only on interfaces required by approved clients:

```conf
listen_addresses = '10.0.5.15'
```

For local-only deployments, prefer Unix sockets or loopback. Do not expose port 5432 directly to untrusted networks.

### 5.2 Defense in depth

Enforce connectivity at multiple layers:

- Cloud security groups or network security groups
- Host firewall
- Kubernetes NetworkPolicy or service-mesh policy
- Private routing and peering
- `pg_hba.conf`
- Database and role authorization

Prefer source workload identities or security-group references over broad CIDR ranges where the platform supports them. Account for IPv4 and IPv6. Validate that container, overlay, or load-balancer networking does not bypass host rules.

Avoid generic `iptables` or `ufw` copy-paste commands without a complete, platform-specific ruleset, persistence plan, rollback procedure, and administrative-access test.

### 5.3 Connection poolers

A pooler can reduce backend connection pressure, but it is not a complete DoS control and does not eliminate application credentials.

When using PgBouncer or another pooler:

- Protect both client-to-pooler and pooler-to-database connections.
- Use separate pooler administration credentials.
- Limit clients, pools, databases, users, queue time, and query time.
- Restrict the pooler's network access and operating-system privileges.
- Keep credentials in an approved secret store.
- Monitor pool exhaustion, queue depth, authentication failure, and backend churn.
- Test rotation and failover.

Transaction pooling can conflict with session state, temporary tables, prepared statements, advisory locks, and session variables. Do not use session-scoped tenant context with transaction pooling unless every transaction sets and validates the context safely.

---

## 6. TLS and Certificate Management

### 6.1 Server configuration

```conf
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_min_protocol_version = 'TLSv1.2'
```

Validate supported protocol and cipher parameters for the exact PostgreSQL and TLS-library versions. Avoid hardcoding a tiny cipher list without compatibility and security testing. TLS 1.3 cipher configuration may use different controls from TLS 1.2 and earlier.

### 6.2 Certificate requirements

- Use an organization-approved certificate authority or managed certificate service.
- Put every DNS name used by clients in the certificate SAN extension.
- Protect the server private key with owner-only permissions required by PostgreSQL.
- Keep CA signing keys outside the database host.
- Monitor expiration and automate renewal where possible.
- Define revocation and emergency replacement procedures.
- Test clients with server hostname verification enabled.

Do not publish an incomplete OpenSSL sequence as production PKI guidance. Certificate profiles, SANs, key usages, validity, serial management, revocation, and secure key generation should come from the organization's PKI process.

### 6.3 Client verification

Clients should use hostname-verifying TLS, such as `sslmode=verify-full`, with a trusted CA bundle appropriate to the environment.

Do not embed passwords in connection URLs. Use an approved secret store, protected password file, service definition, workload identity, or token mechanism supported by the client and platform.

### 6.4 Verify active TLS

Authorized operators can correlate session and TLS state:

```sql
SELECT
  a.datname,
  a.usename,
  a.client_addr,
  s.ssl,
  s.version,
  s.cipher,
  s.bits,
  s.client_dn
FROM pg_stat_activity AS a
JOIN pg_stat_ssl AS s ON s.pid = a.pid;
```

Restrict access to activity and connection metadata according to operational need.

---

## 7. Row-Level Security and Tenant Isolation

### 7.1 RLS is defense in depth

Row-level security can enforce tenant filters independently of application query text, but it must be designed carefully:

- Superusers and roles with `BYPASSRLS` bypass policies.
- Table owners normally bypass RLS unless it is forced.
- Missing or attacker-controlled tenant context can defeat a poorly designed policy.
- Connection pooling can leak session context.
- Policies must cover every applicable command, not only `SELECT`.
- Foreign keys, functions, views, error messages, and timing can create indirect disclosure paths.

Application roles must not own tenant tables and must not have `BYPASSRLS`.

### 7.2 Example tenant policy

```sql
ALTER TABLE app.agent_embeddings ENABLE ROW LEVEL SECURITY;
ALTER TABLE app.agent_embeddings FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_select ON app.agent_embeddings
  FOR SELECT
  TO agent_runtime
  USING (
    tenant_id = NULLIF(current_setting('app.current_tenant', true), '')::uuid
  );

CREATE POLICY tenant_insert ON app.agent_embeddings
  FOR INSERT
  TO agent_runtime
  WITH CHECK (
    tenant_id = NULLIF(current_setting('app.current_tenant', true), '')::uuid
  );
```

Create separate policies for `SELECT`, `INSERT`, `UPDATE`, and `DELETE` according to the workload. Test missing, malformed, and cross-tenant context.

### 7.3 Bind context to trusted identity

Do not let an untrusted client choose an arbitrary tenant by issuing `SET app.current_tenant = ...`.

Use one of these patterns:

- A distinct database login per tenant with policies based on `current_user`
- A trusted transaction broker that derives tenant identity from authenticated claims
- A carefully reviewed `SECURITY DEFINER` entry point that validates authorization before setting transaction-local context
- Separate databases or clusters for stronger isolation

For pooled connections, set context with `SET LOCAL` inside the same transaction as the protected query and ensure the transaction always ends cleanly:

```sql
BEGIN;
SET LOCAL app.current_tenant = '00000000-0000-0000-0000-000000000000';
-- Execute authorized statements.
COMMIT;
```

The example UUID is a placeholder, not an authorization decision. The trusted broker must supply a tenant already authorized for the authenticated principal.

### 7.4 RLS validation

Automate tests for:

- Table owner, application, migration, reporting, and support roles
- Missing, invalid, and unauthorized tenant context
- Every command type
- Views, functions, joins, vector similarity queries, and aggregates
- COPY and bulk operations
- Pool reuse and transaction failure
- Prepared statements and parallel queries where applicable
- Error-message and metadata leakage

For mutually untrusted tenants or high-impact data, consider separate databases or clusters rather than relying on RLS alone.

---

## 8. Secure Functions and Extension Boundaries

### 8.1 `SECURITY DEFINER`

Avoid `SECURITY DEFINER` unless a narrow privilege elevation is required. For every such function:

- Assign an owner that holds only the required privilege.
- Fully qualify referenced objects.
- Set a safe `search_path` that excludes schemas writable by untrusted roles.
- Validate all arguments.
- Revoke default execution from `PUBLIC`.
- Grant execution only to approved callers.
- Avoid dynamic SQL or quote it safely.
- Audit and test the function as privileged code.

Illustrative pattern:

```sql
CREATE FUNCTION trusted.perform_bounded_action(p_id uuid)
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = pg_catalog, trusted, pg_temp
AS $$
BEGIN
  -- Fully qualified, validated, narrowly scoped logic.
  PERFORM trusted.validate_action(p_id);
END;
$$;

REVOKE ALL ON FUNCTION trusted.perform_bounded_action(uuid) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION trusted.perform_bounded_action(uuid) TO agent_runtime;
```

Do not use a generic privileged password-changing function as a hardening example.

### 8.2 Extensions

- Install only required extensions from approved sources.
- Record extension owner, version, package provenance, and privileges.
- Test extension updates with the PostgreSQL version and backup/restore process.
- Restrict who can install, update, or remove extensions.
- Review extension functions that access files, networks, native code, or elevated privileges.
- Remove unused extensions only after dependency and recovery analysis.

Bundled extensions and third-party extensions have different maintenance and trust models; inventory both.

---

## 9. Query and Resource Governance for AI Workloads

AI agents should not receive unrestricted SQL execution against production databases.

Prefer:

- Predefined, parameterized operations
- Stored queries or a query broker
- Read-only replicas for exploration
- Narrow schemas and views
- Row and result limits
- Explicit vector-search tenant predicates
- Separate read and write credentials

Enforce:

- Statement, lock, and idle transaction timeouts
- Per-role connection limits
- Pool and queue limits
- Query result-size limits in the application or proxy
- Cost and concurrency budgets
- Read-only transactions where appropriate
- Cancellation and circuit-breaking behavior

Keyword filtering or a regular expression that permits statements beginning with `SELECT` is not a SQL security boundary.

Monitor expensive vector searches, missing tenant filters, sequential scans over sensitive tables, unexpectedly high result counts, and repeated retries.

---

## 10. Data Protection at Rest

### 10.1 Storage encryption

Use platform-supported encryption for database volumes, snapshots, replicas, and backups. Keep key administration separate from routine database administration where practical.

Managed-service encryption terminology and configuration differ by provider and service. Validate whether encryption is default, configurable only at creation, inherited by replicas, and applicable to backups and exports.

Storage encryption protects lost media and snapshots. It does not protect against a live process or database role that can read the plaintext data.

### 10.2 Filesystem encryption changes

Do not use generic `cryptsetup luksFormat`, filesystem creation, or data-directory migration commands as copy-paste hardening instructions. Formatting a device is destructive, and a safe migration depends on platform, filesystem, service manager, mandatory-access-control labels, mount policy, backup, downtime, rollback, and secure disposal requirements.

A storage-encryption migration requires:

- Verified backups and tested restore
- Approved change and rollback plan
- Correct device identification
- Maintenance window and application shutdown
- Integrity validation before cutover
- Persistent, secure boot-time key handling
- Ownership, permissions, SELinux/AppArmor, and mount options
- Verification after restart
- Approved handling of the old unencrypted data

Use the operating-system or cloud-provider procedure for the exact platform.

### 10.3 Data checksums

Enable data checksums for new clusters where supported. Some versions also support offline checksum enablement or verification tools for existing clusters. Validate the exact procedure and downtime requirements.

Checksums detect certain forms of page corruption; they do not provide confidentiality, authentication, or a replacement for backups.

### 10.4 Field-level encryption

Never place an encryption key literal in SQL text. It may be exposed through application logs, statement logs, activity views, debugging, history, monitoring, or memory.

Prefer application-layer envelope encryption:

1. Authenticate the workload.
2. Obtain or unwrap a narrowly authorized data-encryption key through KMS or HSM policy.
3. Encrypt before sending data to PostgreSQL.
4. Store ciphertext, algorithm/version metadata, and a wrapped-key reference.
5. Restrict and audit decrypt operations.
6. Define key rotation and re-encryption procedures.

Database-side cryptographic functions may be appropriate for specific designs, but key delivery, logging exposure, administrator access, backups, replication, and rotation must be threat-modeled.

### 10.5 Application passwords

Application-user passwords should normally be hashed in the application using an approved password-hashing library and current organizational parameters, then stored as verifier data. Do not conflate PostgreSQL role password hashing with application-user password storage.

If database cryptographic functions are considered, evaluate algorithm support, parameter tuning, input limits, migration, and exposure of plaintext to the database session. Do not put real passwords in documentation examples.

---

## 11. Logging, Auditing, and Monitoring

### 11.1 Baseline logging

Illustrative settings, subject to version and workload validation:

```conf
logging_collector = on
log_connections = on
log_disconnections = on
log_statement = 'ddl'
log_line_prefix = '%m [%p] %q%u@%d %r '
```

Configure failed-authentication logging according to the exact PostgreSQL version and platform. Do not assume every parameter exists in every supported release.

### 11.2 Protect sensitive data in logs

`log_statement = 'all'`, broad duration logging, and detailed audit extensions can capture personal data, credentials, tokens, encryption material, or proprietary queries. Before enabling them:

- Define the audit objective.
- Minimize statement and parameter capture.
- Review version-specific parameter-length controls.
- Redact secrets before they reach SQL.
- Restrict log access and administration.
- Encrypt logs in transit and at rest.
- Protect integrity and time synchronization.
- Define retention and deletion.
- Load-test performance and storage impact.

Do not enable full statement logging merely by labeling an environment “high security.”

### 11.3 pgAudit

If pgAudit is required:

- Confirm compatibility with the PostgreSQL package and version.
- Add it without overwriting other required preload libraries.
- Follow required restart and `CREATE EXTENSION` procedures.
- Select only necessary audit classes.
- Test volume, latency, failure, and sensitive-data exposure.
- Forward records to independently controlled storage.

Installing pgAudit does not by itself establish regulatory compliance.

### 11.4 Detection use cases

Alert on:

- Superuser or emergency-account use
- Role, membership, ownership, or privilege changes
- Authentication spikes and unusual source locations
- New or altered `pg_hba.conf`, TLS, logging, or preload settings
- Extension installation or update
- RLS policy or table-owner changes
- `BYPASSRLS`, replication, or database-creation grants
- Unexpected COPY, bulk reads, or exports
- Large vector or document retrievals
- Access outside normal tenant or service patterns
- Disabled logging or gaps in expected telemetry
- Connection-pool exhaustion and repeated timeouts
- Backup, restore, and replica changes

Correlate PostgreSQL logs with host, container, identity, firewall, cloud, KMS, proxy, and application telemetry.

---

## 12. Backup, Recovery, and Availability

### 12.1 Backup security

- Encrypt backups and replication traffic.
- Use separate, least-privilege backup identities.
- Restrict backup storage and deletion.
- Maintain immutable or logically isolated recovery copies where required.
- Record database, WAL, encryption-key, and software-version dependencies.
- Monitor backup success, size, duration, and unexpected access.
- Test restoration in an isolated environment.

### 12.2 Recovery objectives

Define and test:

- Recovery point objective
- Recovery time objective
- Point-in-time recovery window
- Regional or site failure procedure
- Key-loss and certificate-expiration recovery
- Corruption and ransomware recovery
- Extension and major-version compatibility

High availability is not a backup. Replication can reproduce deletion, corruption, or malicious changes.

### 12.3 Integrity checks

Schedule platform-appropriate checks for:

- Backup restorability
- Page corruption and checksum failures
- Replica lag and divergence
- Index integrity where risk warrants it
- Storage and filesystem errors
- Certificate and key expiration

Respond to integrity failures through a documented incident and recovery process.

---

## 13. Patch and Change Management

### 13.1 PostgreSQL updates

Track PostgreSQL security notices and supported release schedules. Apply supported minor releases under a risk-based service-level objective rather than waiting for an arbitrary quarterly date. Internet exposure, known exploitation, privilege impact, and data sensitivity should shorten remediation timelines.

Test:

- Application and driver compatibility
- Connection poolers and proxies
- Replication and failover
- Extensions
- Backup and restore
- Authentication and TLS
- RLS and authorization regression
- Monitoring and audit output

### 13.2 Major upgrades

Major-version changes require compatibility assessment, extension validation, migration rehearsal, performance testing, rollback planning, and post-upgrade security review.

### 13.3 Configuration management

Store approved PostgreSQL, pooler, firewall, and monitoring configurations in controlled version management without secrets. Detect drift and require review for changes to:

- Network listeners and `pg_hba.conf`
- Roles, memberships, owners, and default privileges
- RLS policies
- TLS and certificates
- Extensions and preload libraries
- Logging and audit settings
- Timeouts and resource limits
- Backup and replication configuration

---

## 14. Incident Response

If compromise is suspected:

1. Activate the incident-response process and preserve the timeline.
2. Restrict access at independently controlled network layers.
3. Preserve PostgreSQL, operating-system, proxy, identity, KMS, and cloud evidence.
4. Revoke or rotate exposed credentials from a known-clean administrative environment.
5. Review active sessions, prepared transactions, replication slots, subscriptions, jobs, functions, extensions, triggers, roles, and ownership.
6. Review data access, exports, schema changes, RLS changes, and logging gaps.
7. Determine whether backups, replicas, or encryption keys were affected.
8. Rebuild or restore from trusted artifacts when integrity cannot be established.
9. Apply patches and corrected controls.
10. Validate authentication, authorization, TLS, RLS, audit, backup, and recovery before restoration.

Do not rotate credentials through a compromised host, and do not assume that a process restart removes persistence or restores data integrity.

---

## 15. Production Security Gate

### Identity and access

- [ ] PostgreSQL runs as a dedicated non-root operating-system user.
- [ ] Human and workload logins are distinct and attributable.
- [ ] Applications do not use superuser, owner, migration, or `BYPASSRLS` roles.
- [ ] Login roles inherit only approved `NOLOGIN` permission roles.
- [ ] Administrative and emergency access is controlled and audited.
- [ ] Credentials are stored, rotated, and revoked through approved mechanisms.

### Authentication and TLS

- [ ] `pg_hba.conf` is narrow, ordered, reviewed, and tested for allows and denials.
- [ ] `trust` and new MD5 password verifiers are absent from production policy.
- [ ] Server identity and hostname are verified by clients.
- [ ] Client-certificate identity mapping is tested where certificate auth is used.
- [ ] Certificate expiration, renewal, revocation, and emergency replacement are monitored.
- [ ] Passwords are absent from SQL scripts, connection URLs, and documentation.

### Network and pooling

- [ ] Database listeners are private and limited to required interfaces.
- [ ] Firewalls and cloud controls permit only approved workloads.
- [ ] IPv4, IPv6, container, and overlay paths are considered.
- [ ] Pooler client and server paths are authenticated and encrypted.
- [ ] Pool, queue, connection, and timeout limits are tested.
- [ ] Pooling mode is compatible with tenant context and application session behavior.

### Authorization and tenancy

- [ ] Schema, table, sequence, function, and default privileges are reviewed.
- [ ] RLS is enabled and forced where required.
- [ ] Application roles do not own protected tables or bypass RLS.
- [ ] Tenant context derives from trusted identity and cannot be self-selected.
- [ ] Cross-tenant tests cover every command, view, function, vector query, and pool-reuse scenario.
- [ ] Stronger database or cluster separation is used where RLS is insufficient.

### Query and resource controls

- [ ] AI workloads use bounded, parameterized operations or an approved query broker.
- [ ] Read and write identities are separated.
- [ ] Statement, lock, idle transaction, connection, and pool limits are set per workload.
- [ ] Expensive vector queries, bulk reads, retries, and result size are monitored.
- [ ] Administrative and migration workloads have separately tested limits.

### Data protection

- [ ] Database volumes, snapshots, replicas, WAL, and backups are encrypted as required.
- [ ] Encryption keys are absent from SQL text and logs.
- [ ] Field encryption uses an approved KMS or HSM design where required.
- [ ] Data checksums and integrity monitoring are configured where supported.
- [ ] Backup restore and point-in-time recovery are tested.
- [ ] Recovery objectives and key-loss procedures are documented.

### Functions, extensions, and supply chain

- [ ] `SECURITY DEFINER` functions have safe ownership, search paths, qualification, and grants.
- [ ] Extensions are approved, inventoried, patched, and privilege-reviewed.
- [ ] PostgreSQL packages or images come from approved, verified sources.
- [ ] Configuration drift and unauthorized changes are detected.

### Logging and response

- [ ] Connection, authentication, role, DDL, RLS, extension, and export events are monitored.
- [ ] Statement and audit logging have been reviewed for secret and personal-data exposure.
- [ ] Logs are centralized, protected, retained, and time-synchronized.
- [ ] Incident containment, clean credential rotation, restore, and rebuild have been exercised.
- [ ] Critical findings block deployment; high findings require remediation or an approved time-bounded exception.

---

## 16. Minimum Priorities

### Required before production

1. Private network access and narrow `pg_hba.conf`
2. Verified TLS and protected credentials
3. Distinct least-privilege workload roles
4. Safe schema and default privileges
5. Trusted tenant binding and tested RLS where applicable
6. Per-workload timeouts, connection limits, and pool controls
7. Encrypted, tested backups and recovery
8. Protected audit logs and security alerts
9. Supported patched versions and approved extensions
10. Tested incident containment and credential rotation

### Required before high-risk AI or multi-tenant use

1. Bounded query interface rather than unrestricted SQL
2. Separate read, write, migration, and administration paths
3. Automated cross-tenant and pool-reuse tests
4. Stronger database or cluster isolation where needed
5. Per-tenant quotas and export monitoring
6. Independent security assessment

---

## Conclusion

PostgreSQL hardening is an ongoing operating model rather than a one-time configuration exercise. Production security requires narrow identities, verified channels, private connectivity, database-enforced authorization, careful pooling, protected keys, bounded queries, recoverable data, observable activity, and controlled change. AI agents should be treated as potentially adversarial clients whose authority and resource use are constrained independently by the database and surrounding platform.
