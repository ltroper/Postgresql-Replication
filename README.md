# **Setting Up a HA Database with PostgreSQL and repmgr**

### **Overview**

We’ll deploy a primary and a standby PostgreSQL instance on two Ubuntu Linodes and configure them for high availability using `repmgr`. `repmgr` is a popular tool for managing replication and failover, allowing seamless promotion of standby nodes if the primary node fails.

---

### **1. Deploy and Prepare the Ubuntu Linodes**

1. **Update the System and Install Required Packages**
   - **Command**:
     ```bash
     sudo apt update && sudo apt upgrade -y
     ```
   - **Explanation**: Before installing any software, update the package lists and apply any available upgrades to keep your system secure and up-to-date. This ensures that dependencies are also updated.

2. **Add PostgreSQL’s Package Repository and Install PostgreSQL with repmgr**
   - **Command**:
     ```bash
     curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /usr/share/keyrings/postgresql-keyring.gpg
     echo "deb [signed-by=/usr/share/keyrings/postgresql-keyring.gpg] http://apt.postgresql.org/pub/repos/apt lunar-pgdg main" | sudo tee /etc/apt/sources.list.d/postgresql.list
     sudo apt update
     sudo apt install postgresql postgresql-16-repmgr -y
     ```
   - **Explanation**: We’re adding PostgreSQL’s official package repository so we can install the latest version. `repmgr` is bundled with PostgreSQL to handle replication and failover management. After adding the repository, we update the package list again to include the new entries and install both PostgreSQL and `repmgr`.

---

### **2. Configure PostgreSQL on the Primary Node**

1. **Modify PostgreSQL Settings**
   - **File**: `/etc/postgresql/16/main/postgresql.conf`
   - **Changes**:
     ```plaintext
     listen_addresses = '*'
     wal_level = replica
     archive_mode = on
     archive_command = '/bin/true'
     max_wal_senders = 10
     max_replication_slots = 10
     hot_standby = on
     shared_preload_libraries = 'repmgr'
     wal_log_hints = on
     ```
   - **Explanation**:
     - `listen_addresses = '*'`: Allows PostgreSQL to accept connections from any IP address, which is essential for replication across different servers.
     - `wal_level = replica`: Configures Write-Ahead Logging (WAL) to support replication.
     - `archive_mode = on` and `archive_command = '/bin/true'`: Prepares PostgreSQL to archive WAL files, which is useful for point-in-time recovery and ensures consistent replication.
     - `max_wal_senders = 10` and `max_replication_slots = 10`: Allow multiple standby nodes by setting the maximum number of connections for WAL replication and slots.
     - `hot_standby = on`: Enables queries on standby nodes while they’re replicating, allowing them to serve read-only requests.
     - `shared_preload_libraries = 'repmgr'`: Loads `repmgr` as a PostgreSQL extension for handling replication.
     - `wal_log_hints = on`: Provides hints for easier recovery in case of corruption, required for certain types of replications.

2. **Configure Client Authentication**
   - **File**: `/etc/postgresql/16/main/pg_hba.conf`
   - **Add the Following**:
     ```plaintext
     local   replication     repmgr                                  trust
     host    replication     repmgr          127.0.0.1/32            trust
     host    replication     repmgr          [secondary IP]/32       trust
     host    all             all             [primary IP]/32         trust
     host    all             all             [secondary IP]/32       trust
     host    all             all             127.0.0.1/32            trust
     ```
   - **Explanation**: This file controls access to PostgreSQL. The above lines allow the `repmgr` user to perform replication-related activities without a password. Trusting specific IPs facilitates secure, passwordless replication between the nodes. Adjust `[primary IP]` and `[secondary IP]` as needed.

3. **Create the `repmgr` User and Database**
   - **Commands**:
     ```bash
     sudo -i -u postgres
     createuser -s repmgr
     createdb repmgr -O repmgr
     psql
     ALTER USER repmgr WITH PASSWORD 'password';
     ALTER USER repmgr WITH REPLICATION;
     ```
   - **Explanation**:
     - We create a superuser named `repmgr` and assign a dedicated database for replication management.
     - Setting a password for `repmgr` and granting replication privileges allows this user to manage the replication setup and act as an administrator for `repmgr`.

4. **Set Up SSH Key-Based Authentication** for Node Communication
   - **Generate SSH Key on Each Node**:
     ```bash
     ssh-keygen -t rsa -b 4096
     ```
   - **Explanation**: Generate SSH keys for secure, passwordless access between the nodes. You’ll need to copy each node’s SSH key into the other’s `authorized_keys` file:
     - On the `primary` node, add the `secondary` node’s public key to `~/.ssh/authorized_keys`.
     - On the `secondary` node, add the `primary` node’s public key to `~/.ssh/authorized_keys`.

---

### **3. Configure repmgr on the Primary Node**

1. **Edit `repmgr.conf` on the Primary Node**
   - **File**: `/etc/repmgr.conf`
   - **Configurations**:
     ```plaintext
     node_id=1
     node_name=pg1
     conninfo='host=[primary IP] user=repmgr dbname=repmgr password=password connect_timeout=2'
     data_directory='/var/lib/postgresql/16/main'
     failover=automatic
     promote_command='repmgr standby promote'
     follow_command='repmgr standby follow'
     log_file='/var/log/postgresql/repmgr.log'
     ```
   - **Explanation**:
     - `node_id=1` and `node_name=pg1`: Identifies this node as the primary.
     - `conninfo`: Connection information for the `repmgr` user to manage this node.
     - `failover=automatic`: Enables automatic failover to a standby node if the primary node fails.
     - `promote_command` and `follow_command`: Commands used by `repmgr` to promote a standby to primary and synchronize standby nodes.
     - `log_file`: Log location for `repmgr` activities.

2. **Register the Primary Node**:
   - **Command**:
     ```bash
     sudo -i -u postgres
     repmgr -f /etc/repmgr.conf primary register
     ```
   - **Explanation**: This command registers the current node (`pg1`) as the primary in the `repmgr` cluster, which is the first step in establishing a managed replication setup.

---

### **4. Configure PostgreSQL and repmgr on the Secondary Node**

1. **Edit repmgr.conf on the Secondary Node**
   - **File**: `/etc/repmgr.conf`
   - **Configuration**:
     ```plaintext
     node_id=2
     node_name=pg2
     conninfo='host=[secondary IP] user=repmgr dbname=repmgr password=password connect_timeout=2'
     data_directory='/var/lib/postgresql/16/main'
     failover=automatic
     promote_command='repmgr standby promote'
     follow_command='repmgr standby follow'
     log_file='/var/log/postgresql/repmgr.log'
     ```
   - **Explanation**: Similar to the primary, but with `node_id=2` and `node_name=pg2` to identify this node as the secondary.

2. **Prepare PostgreSQL Data Directory**
   - **Commands**:
     ```bash
     systemctl stop postgresql
     rm -rf /var/lib/postgresql/16/main/
     mkdir /var/lib/postgresql/16/main/
     chown postgres:postgres /var/lib/postgresql/16/main/
     ```
   - **Explanation**: Remove any pre-existing data in the data directory to ensure a clean environment for cloning the primary’s data.

---

### **5. Clone the Primary Database to the Secondary Node**

1. **Clone Primary to Secondary**:
   - **Command**:
     ```bash
     sudo -i -u postgres
     repmgr -h [primary IP] -U repmgr -d repmgr -f /etc/repmgr.conf standby clone
     ```
   - **Explanation**: This copies the primary node’s data to the secondary node, ensuring they start with the same data. `repmgr` handles this cloning, including all necessary configurations for standby replication.

2. **Register the Standby Node**:
   - **Command**:
     ```bash
     sudo -i -u postgres
     repmgr -f /etc/repmgr.conf standby register
     ```

---

### **6. Verify the Cluster Setup**

1. **Check Cluster Status**:
   - **Command**:
     ```bash
     repmgr -f /etc/repmgr.conf cluster show
     ```
   - **Explanation**: This

 command shows the current status of all nodes in the cluster, allowing you to verify the replication is active and both nodes are communicating.

2. **Simulate Failover**:
   - **Command**:
     ```bash
     repmgr -f /etc/repmgr.conf standby promote
     ```
   - **Explanation**: You can test a failover by promoting the standby to primary. This helps ensure the failover process works as expected in an actual failure event.

---

This setup provides a resilient PostgreSQL environment ready to handle node failures with automated failover, allowing uninterrupted database service.
