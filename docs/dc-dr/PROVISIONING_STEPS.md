# DC-DR Support — Provisioning Steps

## Component Information

| Field | Value |
|-------|-------|
| **Component Name** | DC-DR Support (Data Center — Disaster Recovery) |
| **Purpose** | Provides disaster recovery backup for the primary PostgreSQL database using repmgr, an open-source replication manager. Supports automatic, semi-automatic, and manual failover modes. |
| **Technologies** | PostgreSQL, repmgr (open-source) |

## Prerequisites

### Dependencies

| Dependency | Minimum Version | Purpose |
|------------|-----------------|---------|
| PostgreSQL | 9.6 | Primary and standby database servers |
| repmgr | 4.x | PostgreSQL replication management and failover |
| SSH (passwordless) | — | Secure communication between primary and standby servers |
| rsync | — | File-based replication support |

> **Note:** repmgr must be installed on both the primary and standby servers. Ensure the same major version of PostgreSQL is used on both servers.

### Hardware Requirements

| Role | CPU | RAM | Disk | Notes |
|------|-----|-----|------|-------|
| Primary Server | 4+ cores | 8 GB+ | Sized for production data | Serves all write and read traffic in normal operation |
| Standby Server | 4+ cores | 8 GB+ | Same as primary | Must have sufficient disk to hold a full replica |

### Network Requirements

- Low-latency, high-bandwidth network link between primary and standby servers.
- If primary and standby are in different data centers, ensure a dedicated or VPN link.

## Network Ports and Firewall Restrictions

| Port | Protocol | Direction | Source | Destination | Purpose | Required |
|------|----------|-----------|--------|-------------|---------|----------|
| `5432` | TCP | Bidirectional | Primary Server | Standby Server | PostgreSQL streaming replication | Yes |
| `5432` | TCP | Inbound | App Nodes | Primary Server | Application database connections | Yes |
| `22` | TCP | Bidirectional | Primary Server | Standby Server | SSH for repmgr commands and rsync | Yes |
| `<repmgrd_port>` | TCP | Inbound | Monitoring | Primary & Standby | repmgrd daemon health monitoring | Optional |

> **Note:** Replace `<repmgrd_port>` with your configured monitoring port.

## Provisioning Steps

### Step 1 — Install repmgr on Both Servers

Install repmgr on the primary and standby PostgreSQL servers.

```bash
# On both primary and standby servers
sudo apt-get update
sudo apt-get install -y postgresql-<version>-repmgr

# Verify installation
repmgr --version
```

**Expected outcome:** repmgr is installed and reports its version on both servers.

### Step 2 — Configure PostgreSQL on the Primary Server

Enable streaming replication settings on the primary.

```bash
# Edit postgresql.conf on the primary server
sudo vi /etc/postgresql/<version>/main/postgresql.conf
```

Add or update the following settings:

```ini
# Replication settings
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
hot_standby = on
archive_mode = on
archive_command = '/bin/true'
shared_preload_libraries = 'repmgr'
```

Update `pg_hba.conf` to allow replication connections:

```
# Allow repmgr and replication connections
host    repmgr          repmgr      <standby_ip>/32    trust
host    replication     repmgr      <standby_ip>/32    trust
host    repmgr          repmgr      <primary_ip>/32    trust
host    replication     repmgr      <primary_ip>/32    trust
```

Restart PostgreSQL:

```bash
sudo systemctl restart postgresql
```

**Expected outcome:** PostgreSQL restarts successfully with replication settings enabled.

### Step 3 — Create the repmgr User and Database

```bash
# On the primary server
sudo -u postgres createuser -s repmgr
sudo -u postgres createdb repmgr -O repmgr

# Verify
psql -U repmgr -d repmgr -c "SELECT 1;"
```

**Expected outcome:** The `repmgr` user and database are created. The verification query returns successfully.

### Step 4 — Configure repmgr on the Primary Server

Create the repmgr configuration file on the primary.

```bash
sudo vi /etc/repmgr.conf
```

```ini
node_id=1
node_name='<primary_hostname>'
conninfo='host=<primary_hostname> user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/<version>/main'
use_replication_slots=yes
monitoring_history=yes

# Failover settings (choose one mode)
failover=automatic
# failover=manual

promote_command='/usr/bin/repmgr standby promote -f /etc/repmgr.conf --log-to-file'
follow_command='/usr/bin/repmgr standby follow -f /etc/repmgr.conf --log-to-file --upstream-node-id=%n'
```

**Expected outcome:** repmgr configuration file is created with the correct primary node settings.

### Step 5 — Register the Primary Server

```bash
# Register the primary node with repmgr
sudo -u postgres repmgr -f /etc/repmgr.conf primary register

# Verify registration
sudo -u postgres repmgr -f /etc/repmgr.conf cluster show
```

**Expected outcome:** Primary node is registered. `cluster show` displays the primary node with role `primary` and status `running`.

### Step 6 — Set Up Passwordless SSH

Configure passwordless SSH between the primary and standby servers for repmgr operations.

```bash
# On the primary server (as postgres user)
sudo -u postgres ssh-keygen -t rsa -N "" -f /var/lib/postgresql/.ssh/id_rsa

# Copy the public key to the standby server
sudo -u postgres ssh-copy-id postgres@<standby_hostname>

# Test SSH connectivity
sudo -u postgres ssh postgres@<standby_hostname> "echo 'SSH connection successful'"
```

Repeat in the reverse direction (standby → primary).

**Expected outcome:** Passwordless SSH works in both directions between the postgres users.

### Step 7 — Clone the Standby from the Primary

On the standby server, clone the primary database.

```bash
# Stop PostgreSQL on the standby (if running)
sudo systemctl stop postgresql

# Clone the primary
sudo -u postgres repmgr -h <primary_hostname> -U repmgr -d repmgr -f /etc/repmgr.conf standby clone --dry-run

# If dry-run succeeds, perform the actual clone
sudo -u postgres repmgr -h <primary_hostname> -U repmgr -d repmgr -f /etc/repmgr.conf standby clone
```

**Expected outcome:** Dry-run reports no errors. Clone completes successfully, copying data from the primary to the standby.

### Step 8 — Configure repmgr on the Standby Server

Create the repmgr configuration file on the standby.

```bash
sudo vi /etc/repmgr.conf
```

```ini
node_id=2
node_name='<standby_hostname>'
conninfo='host=<standby_hostname> user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/<version>/main'
use_replication_slots=yes
monitoring_history=yes

# Failover settings (must match primary)
failover=automatic
# failover=manual

promote_command='/usr/bin/repmgr standby promote -f /etc/repmgr.conf --log-to-file'
follow_command='/usr/bin/repmgr standby follow -f /etc/repmgr.conf --log-to-file --upstream-node-id=%n'
```

**Expected outcome:** repmgr configuration is created on the standby with `node_id=2`.

### Step 9 — Start and Register the Standby Server

```bash
# Start PostgreSQL on the standby
sudo systemctl start postgresql

# Register the standby node
sudo -u postgres repmgr -f /etc/repmgr.conf standby register

# Verify the cluster
sudo -u postgres repmgr -f /etc/repmgr.conf cluster show
```

**Expected outcome:** Standby is running and registered. `cluster show` displays both nodes: primary (running) and standby (running, streaming).

### Step 10 — Configure the Failover Mode

Choose and configure the appropriate failover mode.

#### Automatic Failover

```bash
# On the standby server, start the repmgrd daemon
sudo -u postgres repmgrd -f /etc/repmgr.conf --daemonize

# Verify repmgrd is running
ps aux | grep repmgrd
```

The daemon monitors the primary and automatically promotes the standby if the primary becomes unreachable.

#### Semi-Automatic Failover

Set `failover=manual` in `/etc/repmgr.conf` on both servers, but still run `repmgrd` for monitoring. When the primary fails, manually approve the promotion:

```bash
# On the standby server, after confirming primary failure
sudo -u postgres repmgr standby promote -f /etc/repmgr.conf
```

#### Manual Failover

Set `failover=manual` in `/etc/repmgr.conf` and do not run the `repmgrd` daemon. Perform all failover operations manually:

```bash
# 1. Confirm primary is down
sudo -u postgres repmgr -f /etc/repmgr.conf cluster show

# 2. Promote the standby
sudo -u postgres repmgr standby promote -f /etc/repmgr.conf

# 3. Verify the new primary
sudo -u postgres repmgr -f /etc/repmgr.conf cluster show
```

**Expected outcome:** Failover mode is configured according to the chosen strategy.

### Step 11 — Update Application Configuration for Failover

Configure the application to handle database failover.

```bash
cd <app_install_dir>

# Add standby connection information
sed -i 's|^db.standby.url=.*|db.standby.url=jdbc:postgresql://<standby_hostname>:5432/<db_name>|' conf/db.properties
sed -i 's/^db.failover.enabled=.*/db.failover.enabled=true/' conf/db.properties
```

**Expected outcome:** Application is configured to fail over to the standby database when the primary is unavailable.

## Validation

### Validation Class Files

| Class | Purpose |
|-------|---------|
| `ReplicationStatusValidator` | Verifies streaming replication is active between primary and standby |
| `ReplicationLagValidator` | Checks that replication lag is within acceptable thresholds |
| `FailoverReadinessValidator` | Confirms the standby can be promoted (repmgr standby switchover --dry-run) |
| `SSHConnectivityValidator` | Tests passwordless SSH between primary and standby |
| `ClusterTopologyValidator` | Validates the repmgr cluster topology matches the expected configuration |

### Manual Checks

1. **Verify replication status** — On the primary, run:
   ```bash
   sudo -u postgres repmgr -f /etc/repmgr.conf cluster show
   ```
   Confirm both nodes show `running` status and the standby shows `streaming`.

2. **Check replication lag** — On the standby, run:
   ```sql
   SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
   ```
   Lag should be minimal (seconds).

3. **Test failover (non-production)** — Perform a switchover to verify the process:
   ```bash
   sudo -u postgres repmgr standby switchover -f /etc/repmgr.conf --dry-run
   ```

4. **Verify WAL archiving** — Check that WAL files are being archived:
   ```bash
   ls -la /var/lib/postgresql/<version>/main/pg_wal/
   ```

### Automated Checks

```bash
# Run the DC-DR validation suite
cd <install_dir>
./bin/validate.sh --component dc-dr

# Check replication lag
./bin/validate.sh --check replication-lag --max-lag-seconds 30

# Verify cluster health
./bin/validate.sh --check cluster-health
```

## Troubleshooting

| Issue | Possible Cause | Resolution |
|-------|---------------|------------|
| Standby not streaming | Replication slot not created | Verify `use_replication_slots=yes` and re-clone |
| High replication lag | Network bandwidth insufficient | Check network link between DC and DR sites |
| repmgrd not starting | Configuration file error | Check `/etc/repmgr.conf` syntax and paths |
| Clone fails with permission error | SSH keys not configured | Re-run passwordless SSH setup (Step 6) |
| Automatic failover not triggering | repmgrd not running on standby | Start repmgrd daemon on the standby server |
| Split-brain after failover | Both nodes think they are primary | Shut down the old primary, re-clone it as standby |
| pg_hba.conf rejects replication | Missing replication entry | Add replication entries for both server IPs |

## Screenshots

> **Note:** Add screenshots for the following deployment milestones:

- [ ] `repmgr cluster show` output with both nodes healthy
- [ ] Replication lag query result (near-zero lag)
- [ ] Successful switchover dry-run output
- [ ] repmgrd daemon log showing monitoring activity
- [ ] Application failover test — successful connection to standby
