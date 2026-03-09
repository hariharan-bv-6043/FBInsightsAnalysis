# Multi App — Provisioning Steps

## Component Information

| Field | Value |
|-------|-------|
| **Component Name** | Multi App |
| **Version** | — |
| **Purpose** | Enables horizontal scaling by deploying additional application server nodes in a Parent–Child topology. The parent node manages centralized scheduling and caching while child nodes share the user-request load. |
| **Technologies** | RocksDB (metadata cache), gRPC (inter-node communication) |

## Prerequisites

### Dependencies

| Dependency | Minimum Version | Purpose |
|------------|-----------------|---------|
| Java Runtime Environment (JRE) | 8 | Application runtime |
| PostgreSQL | 9.6 | Primary write database |
| RocksDB | 5.x | Centralized metadata cache (managed by parent node) |
| gRPC runtime libraries | 1.x | Inter-app communication between parent and child nodes |
| C-Store extension | — | Read-optimized columnar database (PostgreSQL extension) |
| Bundled Controller | — | Required for PostgreSQL → C-Store sync (see [Remote Controller](../remote-controller/PROVISIONING_STEPS.md)) |

### Hardware Requirements

| Role | CPU | RAM | Disk | Notes |
|------|-----|-----|------|-------|
| Parent Node | 4+ cores | 8 GB+ | 50 GB+ | Runs TaskEngine and RocksDB cache |
| Child Node | 2+ cores | 4 GB+ | 20 GB+ | Serves user requests only |

### Network Requirements

All nodes must be able to communicate over gRPC and access the shared PostgreSQL database.

## Network Ports and Firewall Restrictions

| Port | Protocol | Direction | Source | Destination | Purpose | Required |
|------|----------|-----------|--------|-------------|---------|----------|
| `<grpc_port>` | TCP | Inbound | Child Node | Parent Node | gRPC: cache access and file replication | Yes |
| `<grpc_port>` | TCP | Outbound | Parent Node | Child Node | gRPC: file push replication | Yes |
| `5432` | TCP | Outbound | All App Nodes | PostgreSQL Server | Database writes | Yes |
| `5432` | TCP | Outbound | All App Nodes | C-Store Server | Database reads | Yes |
| `<app_http_port>` | TCP | Inbound | Load Balancer | All App Nodes | User HTTP/HTTPS requests | Yes |

> **Note:** Replace `<grpc_port>` and `<app_http_port>` with the values configured for your environment.

## Provisioning Steps

### Step 1 — Prepare the Parent Node Environment

Install the required runtime and dependencies on the designated parent machine.

```bash
# Install Java (example for Debian/Ubuntu)
sudo apt-get update
sudo apt-get install -y openjdk-8-jre

# Verify Java installation
java -version
```

**Expected outcome:** Java version 1.8.x is reported.

### Step 2 — Deploy the Application on the Parent Node

Copy the application package to the parent node and configure it as the parent.

```bash
# Create the installation directory
sudo mkdir -p <install_dir>

# Extract the application archive
sudo tar -xzf <app_package>.tar.gz -C <install_dir>

# Set the node role to parent
cd <install_dir>
sed -i 's/^node.role=.*/node.role=parent/' conf/node.properties
```

**Expected outcome:** Application files are extracted and `node.role` is set to `parent` in `conf/node.properties`.

### Step 3 — Configure the RocksDB Cache on the Parent Node

The parent node creates and manages the centralized RocksDB cache. Child nodes access it via gRPC.

```bash
# Set RocksDB data directory
sed -i 's|^rocksdb.data.dir=.*|rocksdb.data.dir=<install_dir>/data/rocksdb|' conf/cache.properties

# Set gRPC listen port for cache service
sed -i 's/^grpc.port=.*/grpc.port=<grpc_port>/' conf/grpc.properties
```

**Expected outcome:** RocksDB data directory and gRPC port are configured.

> **Note:** RocksDB is used for framework-level metadata caching only (e.g., internal routing tables, configuration state). Application-level or product-level caching (e.g., user session data, query result caching) is managed separately by the product and is outside the scope of this component.

### Step 4 — Configure the TaskEngine on the Parent Node

The TaskEngine is a centralized scheduler that runs only on the parent node.

```bash
# Enable TaskEngine
sed -i 's/^taskengine.enabled=.*/taskengine.enabled=true/' conf/taskengine.properties

# Configure database connection for task fetching
sed -i 's|^taskengine.db.url=.*|taskengine.db.url=jdbc:postgresql://<db_host>:5432/<db_name>|' conf/taskengine.properties
sed -i 's/^taskengine.db.user=.*/taskengine.db.user=<db_user>/' conf/taskengine.properties
sed -i 's/^taskengine.db.password=.*/taskengine.db.password=<db_password>/' conf/taskengine.properties
```

**Expected outcome:** TaskEngine is enabled and connected to the PostgreSQL database.

### Step 5 — Start the Parent Node

```bash
cd <install_dir>
./bin/start.sh

# Verify the parent node is running
./bin/status.sh
```

**Expected outcome:** Parent node is running. Status output shows `node.role=parent`, TaskEngine is active, and RocksDB cache is initialized.

### Step 6 — Prepare and Deploy Child Nodes

Repeat for each child node to be added to the cluster.

```bash
# Install Java on the child machine
sudo apt-get update
sudo apt-get install -y openjdk-8-jre

# Create installation directory and extract application
sudo mkdir -p <install_dir>
sudo tar -xzf <app_package>.tar.gz -C <install_dir>

# Set the node role to child
cd <install_dir>
sed -i 's/^node.role=.*/node.role=child/' conf/node.properties

# Point the child to the parent node for gRPC communication
sed -i 's/^parent.host=.*/parent.host=<parent_hostname>/' conf/grpc.properties
sed -i 's/^parent.grpc.port=.*/parent.grpc.port=<grpc_port>/' conf/grpc.properties
```

**Expected outcome:** Child node is configured with `node.role=child` and knows the parent node's gRPC address.

### Step 7 — Configure Database Connections on the Child Node

```bash
# PostgreSQL write connection
sed -i 's|^db.write.url=.*|db.write.url=jdbc:postgresql://<db_host>:5432/<db_name>|' conf/db.properties

# C-Store read connection
sed -i 's|^db.read.url=.*|db.read.url=jdbc:postgresql://<cstore_host>:5432/<db_name>|' conf/db.properties
```

**Expected outcome:** Child node is configured to write to PostgreSQL and read from C-Store.

### Step 8 — Start the Child Node

```bash
cd <install_dir>
./bin/start.sh

# Verify the child node is running and connected to parent
./bin/status.sh
```

**Expected outcome:** Child node is running. Status output shows `node.role=child` and confirms gRPC connection to the parent.

### Step 9 — Configure Shared File System (Optional)

For centralized product file maintenance across all nodes.

```bash
# Mount the shared file system on each node
sudo mount -t nfs <nfs_server>:<shared_path> <install_dir>/shared

# Add to /etc/fstab for persistence
echo "<nfs_server>:<shared_path> <install_dir>/shared nfs defaults 0 0" | sudo tee -a /etc/fstab
```

**Expected outcome:** Shared file system is mounted and accessible from all nodes.

### Step 10 — Configure Leader Election

Leader election ensures that only one parent node is active at any time.

```bash
# Enable leader election
sed -i 's/^leader.election.enabled=.*/leader.election.enabled=true/' conf/cluster.properties

# Set cluster node list
sed -i 's/^cluster.nodes=.*/cluster.nodes=<parent_host>:<grpc_port>,<child1_host>:<grpc_port>,<child2_host>:<grpc_port>/' conf/cluster.properties
```

**Expected outcome:** Leader election is enabled and the cluster node list is configured.

## Validation

### Validation Class Files

| Class | Purpose |
|-------|---------|
| `ClusterHealthValidator` | Validates that all nodes in the cluster are reachable and reporting correct roles |
| `RocksDBCacheValidator` | Verifies RocksDB cache initialization and accessibility from child nodes via gRPC |
| `TaskEngineValidator` | Confirms TaskEngine is running on the parent node and successfully fetching tasks |
| `GRPCConnectionValidator` | Tests gRPC connectivity between parent and child nodes |
| `FileReplicationValidator` | Validates file push/pull replication between nodes |

### Manual Checks

1. **Verify node roles** — Confirm `status.sh` on each node reports the correct role (parent or child).
2. **Test cache access** — From a child node, trigger a cache lookup and verify it is served by the parent's RocksDB instance.
3. **Verify TaskEngine** — Check the parent node logs for successful task fetch and execution entries.
4. **Test file replication** — Place a test file on the parent and verify it is replicated to child nodes.
5. **Load balancer health** — Confirm the load balancer is distributing requests across all app nodes.

### Automated Checks

```bash
# Run the cluster validation suite
cd <install_dir>
./bin/validate.sh --component multi-app

# Check gRPC connectivity from a child node
./bin/validate.sh --check grpc-connection --target <parent_host>:<grpc_port>

# Verify RocksDB cache status
./bin/validate.sh --check rocksdb-cache
```

## Troubleshooting

| Issue | Possible Cause | Resolution |
|-------|---------------|------------|
| Child cannot reach parent via gRPC | Firewall blocking `<grpc_port>` | Open `<grpc_port>` TCP inbound on parent node |
| RocksDB cache not initialized | Missing data directory | Verify `rocksdb.data.dir` path exists and is writable |
| TaskEngine not running on parent | `taskengine.enabled` not set to `true` | Check `conf/taskengine.properties` |
| TaskEngine fails to fetch tasks | Database connection error | Verify `taskengine.db.url`, user, and password settings |
| File replication fails between nodes | gRPC connection timeout | Check network latency and increase `grpc.timeout` in `conf/grpc.properties` |
| Leader election not converging | Incomplete cluster node list | Ensure all node addresses are listed in `cluster.nodes` |
| Child node status shows disconnected | Parent node not running | Start the parent node first, then restart the child |

## Screenshots

> **Note:** Add screenshots for the following deployment milestones:

- [ ] Parent node status output after successful start
- [ ] Child node status output showing gRPC connection to parent
- [ ] RocksDB cache metrics dashboard
- [ ] TaskEngine log showing successful task execution
- [ ] Load balancer dashboard with all nodes healthy
- [ ] Cluster validation suite output (all checks passed)
