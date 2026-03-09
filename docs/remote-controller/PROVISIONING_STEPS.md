# Remote Controller — Provisioning Steps

## Component Information

| Field | Value |
|-------|-------|
| **Component Name** | Remote Controller |
| **Purpose** | A standalone Java tool that captures changes from PostgreSQL via triggers and synchronizes data to C-Store (columnar read database) for analytics workloads. Supports bundled (default) and remote deployment modes. |
| **Technologies** | PostgreSQL, C-Store (columnar extension), Java |

## Prerequisites

### Dependencies

| Dependency | Minimum Version | Purpose |
|------------|-----------------|---------|
| Java Runtime Environment (JRE) | 8 | Controller runtime |
| PostgreSQL | 9.6 | Source write database |
| C-Store extension (cstore_fdw) | — | Target read database (PostgreSQL extension) |
| PostgreSQL triggers (pre-configured) | — | Change capture on write tables |

> **Note:** The Bundled Controller is included with the product by default. Install it before considering a Remote Controller.

### Hardware Requirements

| Deployment Mode | CPU | RAM | Disk | Notes |
|-----------------|-----|-----|------|-------|
| Bundled Controller | Shared with app node | Shared | Shared | Runs on the same machine as the application |
| Remote Controller | 2+ cores | 4 GB+ | 20 GB+ | Dedicated machine for read-load offloading |

### Network Requirements

The controller must have network access to both the source PostgreSQL database and the target C-Store database.

## Network Ports and Firewall Restrictions

| Port | Protocol | Direction | Source | Destination | Purpose | Required |
|------|----------|-----------|--------|-------------|---------|----------|
| `5432` | TCP | Outbound | Controller | PostgreSQL Server | Read change data from source database | Yes |
| `5432` | TCP | Outbound | Controller | C-Store Server | Write synchronized data to target database | Yes |
| `<controller_mgmt_port>` | TCP | Inbound | Admin workstation | Controller | Management / health-check endpoint | Optional |

> **Note:** For bundled mode, the controller connects locally (localhost:5432). For remote mode, ensure firewall rules allow TCP 5432 between the controller machine and both database servers.

## Overview of Controller Types

### Bundled Controller (Default)

- Included with the product installation.
- Runs on the same machine as the application server.
- Manages a separate schema within the same PostgreSQL instance.
- Writes go to PostgreSQL; reads are served from C-Store tables.
- Sync is handled transparently — no additional setup beyond initial provisioning.

### Remote Controller (Optional)

- Deployed on a separate dedicated machine.
- Used when specific workspaces contain RAM-intensive tables or queries.
- Allows splitting read load: map heavy workspaces to the remote controller's C-Store instance.
- Requires explicit workspace-to-controller mapping.

## Provisioning Steps — Bundled Controller

### Step 1 — Verify PostgreSQL and C-Store Extension

Confirm that PostgreSQL is running and the C-Store extension is available.

```bash
# Check PostgreSQL status
sudo systemctl status postgresql

# Connect and verify C-Store extension
psql -U <db_user> -d <db_name> -c "SELECT * FROM pg_available_extensions WHERE name = 'cstore_fdw';"
```

**Expected outcome:** PostgreSQL is active. The `cstore_fdw` extension is listed as available.

### Step 2 — Install the C-Store Extension

```sql
-- Connect to the target database
\c <db_name>

-- Create the extension
CREATE EXTENSION IF NOT EXISTS cstore_fdw;

-- Create the foreign server for C-Store
CREATE SERVER cstore_server FOREIGN DATA WRAPPER cstore_fdw;
```

**Expected outcome:** Extension and foreign server are created without errors.

### Step 3 — Create the Sync Schema

The bundled controller uses a separate schema to manage sync metadata.

```sql
-- Create the controller sync schema
CREATE SCHEMA IF NOT EXISTS controller_sync;

-- Create sync tracking table
CREATE TABLE controller_sync.change_log (
    id BIGSERIAL PRIMARY KEY,
    table_name VARCHAR(255) NOT NULL,
    operation VARCHAR(10) NOT NULL,
    row_data JSONB,
    captured_at TIMESTAMP DEFAULT NOW(),
    synced BOOLEAN DEFAULT FALSE
);
```

**Expected outcome:** Schema and tracking table are created.

### Step 4 — Configure Change-Capture Triggers

Ensure triggers are installed on all tables that need to be synchronized.

```sql
-- Example trigger for a target table
CREATE OR REPLACE FUNCTION controller_sync.capture_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO controller_sync.change_log (table_name, operation, row_data)
    VALUES (TG_TABLE_NAME, TG_OP, row_to_json(NEW)::jsonb);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Attach trigger to a table
CREATE TRIGGER sync_trigger
AFTER INSERT OR UPDATE ON <target_table>
FOR EACH ROW EXECUTE FUNCTION controller_sync.capture_changes();
```

**Expected outcome:** Triggers are installed. Inserts and updates are captured in `controller_sync.change_log`.

### Step 5 — Start the Bundled Controller

```bash
cd <install_dir>

# Configure bundled controller mode
sed -i 's/^controller.mode=.*/controller.mode=bundled/' conf/controller.properties
sed -i 's|^controller.db.url=.*|controller.db.url=jdbc:postgresql://localhost:5432/<db_name>|' conf/controller.properties

# Start the controller
./bin/controller-start.sh

# Verify controller status
./bin/controller-status.sh
```

**Expected outcome:** Bundled controller is running. Status shows `mode=bundled` and `sync.status=active`.

## Provisioning Steps — Remote Controller

### Step 1 — Prepare the Remote Machine

Install Java and create the installation directory on the dedicated remote machine.

```bash
# Install Java
sudo apt-get update
sudo apt-get install -y openjdk-8-jre

# Create installation directory
sudo mkdir -p <install_dir>
```

**Expected outcome:** Java 1.8.x is installed and the directory is created.

### Step 2 — Deploy the Controller Package

```bash
# Copy the controller package to the remote machine
scp <controller_package>.tar.gz <remote_user>@<remote_host>:<install_dir>/

# Extract on the remote machine
ssh <remote_user>@<remote_host> "tar -xzf <install_dir>/<controller_package>.tar.gz -C <install_dir>"
```

**Expected outcome:** Controller files are extracted on the remote machine.

### Step 3 — Configure Remote Controller Mode

```bash
cd <install_dir>

# Set controller mode to remote
sed -i 's/^controller.mode=.*/controller.mode=remote/' conf/controller.properties

# Configure source PostgreSQL connection
sed -i 's|^controller.source.db.url=.*|controller.source.db.url=jdbc:postgresql://<pg_host>:5432/<db_name>|' conf/controller.properties
sed -i 's/^controller.source.db.user=.*/controller.source.db.user=<db_user>/' conf/controller.properties
sed -i 's/^controller.source.db.password=.*/controller.source.db.password=<db_password>/' conf/controller.properties

# Configure target C-Store connection
sed -i 's|^controller.target.db.url=.*|controller.target.db.url=jdbc:postgresql://<cstore_host>:5432/<db_name>|' conf/controller.properties
sed -i 's/^controller.target.db.user=.*/controller.target.db.user=<cstore_user>/' conf/controller.properties
sed -i 's/^controller.target.db.password=.*/controller.target.db.password=<cstore_password>/' conf/controller.properties
```

**Expected outcome:** Remote controller is configured with source (PostgreSQL) and target (C-Store) database connections.

### Step 4 — Map Workspaces to the Remote Controller

Specify which workspaces should have their read load handled by this remote controller.

```bash
# Map specific workspaces
sed -i 's/^controller.workspaces=.*/controller.workspaces=<workspace_id_1>,<workspace_id_2>/' conf/controller.properties
```

**Expected outcome:** The specified workspaces are mapped to this remote controller instance.

### Step 5 — Install C-Store on the Remote Machine

Set up the C-Store database on the remote machine to serve reads.

```bash
# Install PostgreSQL and C-Store extension on the remote machine
sudo apt-get install -y postgresql postgresql-server-dev-all

# Build and install cstore_fdw (or use package manager if available)
# Then in psql:
psql -U <cstore_user> -d <db_name> -c "CREATE EXTENSION IF NOT EXISTS cstore_fdw;"
psql -U <cstore_user> -d <db_name> -c "CREATE SERVER cstore_server FOREIGN DATA WRAPPER cstore_fdw;"
```

**Expected outcome:** C-Store extension is installed and configured on the remote machine.

### Step 6 — Start the Remote Controller

```bash
cd <install_dir>
./bin/controller-start.sh

# Verify controller status
./bin/controller-status.sh
```

**Expected outcome:** Remote controller is running. Status shows `mode=remote`, mapped workspaces, and `sync.status=active`.

### Step 7 — Update Application Configuration

Point the application to use the remote controller's C-Store for mapped workspaces.

```bash
# On the application server, add the remote C-Store endpoint
cd <app_install_dir>
sed -i 's|^db.read.remote.url=.*|db.read.remote.url=jdbc:postgresql://<cstore_host>:5432/<db_name>|' conf/db.properties
sed -i 's/^db.read.remote.workspaces=.*/db.read.remote.workspaces=<workspace_id_1>,<workspace_id_2>/' conf/db.properties
```

**Expected outcome:** Application routes read queries for the specified workspaces to the remote C-Store instance.

## Validation

### Validation Class Files

| Class | Purpose |
|-------|---------|
| `ControllerSyncValidator` | Verifies that the sync process is running and change_log entries are being consumed |
| `CStoreSchemaValidator` | Validates that C-Store foreign tables match the source PostgreSQL schema |
| `TriggerInstallationValidator` | Confirms change-capture triggers are installed on all required tables |
| `WorkspaceMappingValidator` | Validates workspace-to-controller mapping consistency |
| `DataConsistencyValidator` | Compares row counts and checksums between PostgreSQL source and C-Store target |

### Manual Checks

1. **Verify sync status** — Run `controller-status.sh` and confirm `sync.status=active`.
2. **Check change_log consumption** — Query `controller_sync.change_log` and verify that `synced=TRUE` for recent entries.
3. **Compare row counts** — For a mapped table, compare row counts between PostgreSQL and C-Store.
4. **Test a read query** — Execute an analytics query and confirm it reads from C-Store (check query plan with `EXPLAIN`).
5. **Workspace routing** — For remote mode, verify that queries for mapped workspaces are routed to the remote C-Store.

### Automated Checks

```bash
# Run the controller validation suite
cd <install_dir>
./bin/validate.sh --component controller

# Check sync lag
./bin/validate.sh --check sync-lag --max-lag-seconds 60

# Validate data consistency for a specific table
./bin/validate.sh --check data-consistency --table <table_name>
```

## Troubleshooting

| Issue | Possible Cause | Resolution |
|-------|---------------|------------|
| Controller fails to start | Missing database credentials | Verify `controller.properties` connection settings |
| Sync lag increasing | High write volume on source | Increase controller thread pool size in `controller.properties` |
| C-Store table schema mismatch | Source table altered without re-sync | Run schema migration via `controller-migrate.sh` |
| Triggers not capturing changes | Trigger dropped or disabled | Re-run trigger installation script |
| Remote controller cannot reach PostgreSQL | Firewall blocking port 5432 | Open TCP 5432 from controller machine to PostgreSQL server |
| Workspace queries not routed to remote | Missing workspace mapping in app config | Verify `db.read.remote.workspaces` in app `conf/db.properties` |

## Screenshots

> **Note:** Add screenshots for the following deployment milestones:

- [ ] Bundled controller status output after successful start
- [ ] Remote controller status output showing mapped workspaces
- [ ] Change log table showing synced entries
- [ ] Analytics query plan showing C-Store foreign scan
- [ ] Data consistency validation output
