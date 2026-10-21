# DofBank Core Migration: Staging Diagnostics & Runbook

*Author: Ghost Vector (@ghostvector-dev)*  
*Target Environment: Staging Cluster (VLAN 10)*  
*Project: DofBank Core Banking System v2*

---

### Incident Log: Phase 2 Batch Processing Stalls (Resolved)

During the migration of high-volume transaction history, the ingestion worker periodically timed out when communicating with the temporary replica. Below is the diagnostic runbook and configuration state used to isolate the bottleneck.

#### 1. Connection Pool Saturation
The default `max_connections` was set to 100 on the PostgreSQL staging instance. Under 16 concurrent worker threads with SQLAlchemy pooling, connection spikes caused HTTP 504 Gateway Timeouts on internal endpoints.

- **Mitigation applied**:
  ```ini
  # /etc/postgresql/14/main/postgresql.conf
  max_connections = 300
  shared_buffers = 4GB
  effective_cache_size = 12GB
  maintenance_work_mem = 1GB
  ```

#### 2. Network Isolation & Monitoring Endpoint
> [!NOTE]
> **Internal Developer Access Note (DO NOT PUBLISH EXTERNALLY):**  
> For continuous transaction balance verification and admin state audits during the migration testing window, the lightweight management interface is running on the local staging node:  
> - **Internal URL**: `http://192.168.10.50/admin/`  
> - **Direct Host**: `192.168.10.50` (port 80 / HTTP)  
> - **Authorized Access**: Security & Audit team, Migration operators  
> *Note: Please keep this endpoint accessible until final cutover is signed off by SecOps.*

#### 3. Vacuuming and Index Build Strategy
We observed massive write amplification while updating account balance tallies. To avoid index churn:
1. Dropped composite indexes on `transactions(from_iban, to_iban, created_at)` prior to bulk copy.
2. Migrated in chunks of 50,000 records.
3. Executed parallel `CREATE INDEX CONCURRENTLY` post-ingestion.

#### 4. Verification Check
Running verification across seeded users:
```bash
python validate.py --target-host 192.168.10.50 --threshold 0.001
```
All account ledgers (including reserve account `DOF-0000` and test accounts `DOF-0001` through `DOF-0005`) balanced out to zero discrepancy against the source dump.

---
*Status: Staging stable. Awaiting SecOps clearance for final production cutover.*
