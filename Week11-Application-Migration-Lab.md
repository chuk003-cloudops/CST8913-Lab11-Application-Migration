# Tailwind Traders | Week 11 Application Migration Lab

CST8913 Cloud Migration

## Task 1 - Set up discovery tooling

Use **agentless discovery and dependency analysis**. Register two Azure Migrate Discovery and Assessment appliances in one project. Use the VMware appliance for WEB01, WEB02 and APP01 through their shared vCenter, assuming a single vCenter and reachable guest networks. Use the physical-server appliance for SQL01. It needs direct access to the physical Windows host. Add an appliance if a separate vCenter or isolated network cannot be covered by this design. Record LB01 separately from the server inventory and export its listeners, pools, rules, certificates and probes.

| Discovery purpose | Credentials and access to prepare |
| --- | --- |
| VMware source | A dedicated, scoped vCenter account with read access and the guest-operation privileges required for guest discovery. Verify VMware Tools on each VM. |
| Software inventory | Dedicated guest credentials for each applicable operating system. Use the documented minimum Windows permissions or a regular Linux account for inventory. Confirm which hosts and applications they cover. |
| SQL instances and databases | A Windows/domain or SQL account with the documented low-privilege read permissions on SQL01 and its databases. Have the DBA validate access to instance and feature metadata. |
| Dependency mapping | The VMware guest credentials and required guest-operation privileges for VMware, plus supported Windows remoting and scoped host credentials for physical SQL01. Use network and application telemetry for UDP, transient traffic or latency where agentless TCP collection has gaps. |

1. Register the appliances, confirm DNS, clocks, routing, connectivity and credentials, then compare the discovered inventory against the operator's records.
2. Place appliances in restricted network segments, allow only required communication, vault credentials, and check discovery health and credential coverage regularly.
3. Collect one month of representative activity including peak traffic, backup runs, and patch windows. Review telemetry gaps and validate business-critical connections with owners.

## Task 2 - Plan the assessment

| Setting | Selection and reason |
| --- | --- |
| Assessment | Production Azure VM assessment for all four servers, plus an Azure SQL assessment for SQL01. Use production uptime and assess non-production separately if it is later added. |
| Region and history | Canada Central is the proposed primary region, assuming network latency, residency, availability and quota checks pass. Collect one month of performance history to cover regular peak and maintenance cycles. |
| Sizing | Performance-based sizing with the 95th percentile and a 1.3 comfort factor. Check CPU, memory, disk IOPS and throughput, and network use. Compare with the as-is inventory as a sizing cross-check. Reassess if the workload is seasonal or confidence is low. |
| Pricing and licensing | Pay-as-you-go for the baseline and continuous production uptime. Use license-included pricing until Finance verifies Windows and SQL entitlements. Re-estimate with Azure Hybrid Benefit where eligible and evaluate reservations after sizing stabilizes. Include disks, backups, network, monitoring and SQL licensing in the estimate. |

## Task 3 - Analyze dependencies

Inventory WEB01 and WEB02 (VMware frontends), APP01 (VMware API), SQL01 (physical SQL Server 2017), LB01 (internal load balancer), and the firewall, DNS, certificates, backup and monitoring services. The ports below are proposed discovery checks. Validate the actual listener, service account, ACL and direction with the owner before cutover.

| Source and destination | Protocol or service to verify |
| --- | --- |
| Internal clients to LB01 | HTTPS TCP 443, virtual hostname and certificate chain |
| LB01 to WEB01 and WEB02 | HTTPS TCP 443 or the configured backend port, plus health probe and health path |
| WEB01 and WEB02 to APP01 | Internal HTTPS API, expected TCP 443, including API credentials and TLS trust |
| APP01 to SQL01 | TDS over TCP 1433 for a default instance. Record the actual static port and any agreed named-instance discovery need. |
| Application servers to internal DNS | UDP/TCP 53 and split-horizon records |
| Domain-joined hosts to identity services | Kerberos TCP/UDP 88 and LDAPS TCP 636 where used. Verify service accounts, SPNs and delegation. |
| SQL01 to backup storage or backup proxy | Nightly backup channel, such as SMB TCP 445 or the backup vendor's HTTPS endpoint. Confirm destination and restore access. |
| Hosts to approved monitoring endpoints | Agent telemetry over HTTPS TCP 443 and alert ownership |
| Nodes to approved time sources | NTP UDP 123 where configured, especially for identity and audit timestamps |
| Hosts to patch, license or vendor services | Record real FQDNs, ports, proxy rules, licenses, and IP or MAC restrictions |

Normalize exporter results by source, destination, process, protocol and port. Filter repeated probes, known operating-system chatter, self-connections and generic Internet noise after corroborating ownership. Retain required external DNS, monitoring, licensing, patch, backup and identity destinations even when the addresses are outside private ranges. Reconcile short-lived connections against the firewall, SQL Agent, backup schedule and owner interviews.

## Task 4 - Validate with the application owner

This scenario specifies production behavior and a hard one-hour outage limit. The other items below are validation questions and planning assumptions for the owner, infrastructure team and DBA.

| Topic | Owner validation |
| --- | --- |
| Criticality, users and SLA | Confirm business impact and affected users. Get the uptime SLA, RPO target and exact cutover window in writing. Require an outage rehearsal that stays within 60 minutes, including rollback allowance. |
| Data and backup | Confirm SQL data classification and residency with the data owner. Verify nightly backups, retention, encryption, access and a tested restore. Define the allowed transaction-loss threshold. |
| Patching and support | Confirm OS and SQL support dates, approved maintenance slots, patch ownership and reboot restrictions. |
| Licensing, firewall and IP changes | Confirm Windows and SQL license mobility, any IP or MAC-bound licensing, firewalls on each hop, source allowlists, DNS and certificate dependencies. |
| Operations | Confirm alert routing, health probes, service priorities and owners. Walk through every dependency above and require resolution of any undocumented critical path. |

| Tier | Candidate VM size to validate against assessment |
| --- | --- |
| Each WEB server | Standard_D2s_v5 (2 vCPU, 8 GiB). Validate peak concurrency and whether one VM can serve traffic if the other is out. |
| APP01 | Standard_D4s_v5 (4 vCPU, 16 GiB). Validate API latency, garbage collection, CPU and memory under replayed load. |
| SQL01 | Standard_E4ds_v5 (4 vCPU, 32 GiB). Confirm memory, transaction log latency, data and log IOPS, tempdb sizing and disk throughput using measured baselines. |

Replace LB01 with an internal Standard Load Balancer for TCP pass-through, or a private Application Gateway if HTTP routing or TLS termination is required. Match health probes, backend ports, certificate behavior and failover. Validate availability requirements for the web tier and single APP/SQL tiers. Review assessment readiness, disks, capacity and cost before approving these candidate sizes.

- **SQL Server on Azure VM:** Preferred initial rehost if instance compatibility and downtime dominate. Stage and sync a target using backups and log shipping or another rehearsed method. Keep SQL Agent jobs, server objects and cross-database behavior, and retain OS and SQL operational responsibility.
- **Azure SQL Managed Instance:** Assess instance features, network connectivity, jobs and logins. A compatible managed target can reduce operations work, but validate feature compatibility and rehearse a supported online or low-downtime path.
- **Azure SQL Database:** Consider after checking database-level compatibility. Plan changes for instance-wide jobs, logins and cross-database dependencies, and test the application contract before committing to it.

The SLA gate stays open until the owners approve the dependency register and a rehearsal demonstrates the required timing and recovery objectives.

## Task 5 - Migration runbook

1. **Before the window:** Approve the assessment with the owner and DBA. Prepare Azure subnets, private routing to on-premises, NSGs, DNS, secrets, monitoring, backup and the target load balancer. Lower internal DNS TTL in advance and document original DNS, pool and connection settings. Test-migrate the VMware VMs in an isolated network. Pre-stage SQL, verify restored copies and sync lag, and rehearse cutover and rollback. Assign migration lead, DBA, network, application and operations owners.
2. **Stage each server group:** Replicate and smoke-test WEB01 and WEB02 behind the Azure internal load balancer, with health probes active and production traffic held. Stage APP01 on its private subnet. Validate API authentication and private SQL connectivity and stage the new SQL connection string in the approved secret store. Clone SQL jobs, logins and permissions. Disable duplicate scheduled writers until cutover.
3. **Cutover, minutes 0-20:** Stop or drain writes on the old app. Pause scheduled writers. Have the DBA apply the final logs and confirm target consistency and expected recovery point. Keep the source and backups available. Switch the deployed application to the new SQL endpoint and validate authentication and connectivity.
4. **Cutover, minutes 20-40:** Change internal DNS and direct LB01's virtual hostname to the Azure private frontend. Check TTL, record propagation, pool membership, HTTPS, probes and both web backends. Run read and controlled write transactions, data checks, API smoke tests, and error and latency monitoring. Release writers only with owner and DBA approval. Keep the original services available during the observation window.
5. **Back out if a gate fails:** If SQL sync, checks or traffic cannot be verified by minute 40, start rollback and reserve the final 20 minutes. Drain the new frontend, restore the prior DNS and load balancer, revert the connection string and resume the source only after DBA sign-off. If any target writes occurred, reconcile or reverse-sync them before routing production writes back. Escalate when consistency or the time budget is at risk.
6. **After migration:** Check both frontend instances and probes, API latency and auth, SQL consistency and jobs, captured logs, certificates, alerting, firewall reachability, backup and a restore test. Record time offline, recovery point and outstanding issues. Retire the on-premises deployment only after the owner signs off and the rollback retention period ends.

## Task 6 - Plan the waves

| Wave | Servers | Reason and exit gate |
| --- | --- | --- |
| 1 | WEB01, WEB02, and the Azure replacement for LB01 | Stage and test the stateless frontend together with its probes and certificate. Keep existing production routing while verifying private API reachability and failover. |
| 2 | APP01 | Stage the API after the frontend test path exists. Validate both frontend access and the SQL connection over approved routes with synthetic transactions. |
| 3 | SQL01 | Highest data and downtime risk. Pre-seed outside the outage, then complete sync and validate during the shared cutover. Promote all tiers together only after the database, API and web gates pass. |

These waves describe staging order. The shared production cutover keeps the three dependent tiers on a coherent release within the one-hour window.

## References

- CST8913 Week 11 lecture, slides 5-14.
- [Microsoft Learn: VMware assessment support](https://learn.microsoft.com/en-us/azure/migrate/migrate-support-matrix-vmware)
- [Microsoft Learn: Physical-server assessment support](https://learn.microsoft.com/en-us/azure/migrate/migrate-support-matrix-physical)
- [Microsoft Learn: Discovery credentials](https://learn.microsoft.com/en-us/azure/migrate/add-server-credentials)
- [Microsoft Learn: Assessment settings](https://learn.microsoft.com/en-us/azure/migrate/concepts-assessment-overview)
- [Microsoft Learn: SQL Managed Instance migration guide](https://learn.microsoft.com/en-us/azure/azure-sql/migration-guides/managed-instance/sql-server-to-managed-instance-guide)
