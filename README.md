# Podman-dbscaling-mariadbproxysql For Home Lab environment.

Final Plan: 3-Node Server with K3s, Containerized MariaDB (Master-Slave Replication), and ProxySQL (proxmox and mariadb workbench are optional) 

##### Goal: 
A highly available and scalable MariaDB deployment with a dedicated master, two slaves, and ProxySQL for database load balancing, all containerized and orchestrated by K3s.

## Components:

### Hardware:

* Three physical server nodes (Nodes 1, 2, and 3).

* Software Stack (on each node):

* Linux Operating System (e.g., Ubuntu Server, CentOS Stream)

* K3s (lightweight Kubernetes)

* Containerd (or other container runtime, managed by K3s)

* Docker Engine or Podman(necessary for building/managing images if not using buildkit)

### Containerized Applications:

* MariaDB Master: A single Docker or Podman container running MariaDB, configured as the master server in the replication setup.

* MariaDB Slaves: Two Docker or Podman containers running MariaDB, configured as slaves replicating from the master.

* ProxySQL: A Docker or Podman container running ProxySQL to provide load balancing and query routing for the MariaDB cluster.

### Key Points/Confirmations:

* K3s Cluster: You'll have a three-node K3s cluster, providing high availability for the control plane and the ability to schedule your application pods across the nodes.

* Master-Slave Replication: MariaDB will use asynchronous master-slave replication for read scalability and basic HA.

* ProxySQL: A great option for load balancing, query routing, and connection pooling.

* No Proxmox: There is no Proxmox involved; you are installing the OS and K3s directly on the hardware.

### Containerized Everything: 
All the components (MariaDB and ProxySQL) will be running in Docker or podman containers managed by K3s.

### Manual ProxySQL Configuration: 
You understand that you will be manually setting up and managing ProxySQL to manage traffic and to scale resources across your databases

## Detailed Steps:

### Prepare the Servers:

* Install Linux on each server.

* Configure networking (static IP addresses, DNS).

* Harden the operating system (firewall, security updates).

### Install K3s:

* Install K3s on each server.

* Configure K3s for HA using a clustered etcd setup.

* Define Kubernetes Deployments and Services:

* Create Kubernetes Deployment manifests for the MariaDB master, MariaDB slaves, and ProxySQL.

* Create a Kubernetes Service manifest for ProxySQL, exposing it to the outside world.


### Configure MariaDB Replication:

* Enable binary logging on the MariaDB master server.

* Create a replication user on the MariaDB master server.

* Configure the MariaDB slave servers to connect to the master server and start replication.


### Configure ProxySQL:

* Configure ProxySQL to connect to the MariaDB master and slave servers.

* Configure query routing rules to send read queries to the slaves and write queries to the master.

### Deploy the Pods:

* Apply the Kubernetes Deployment and Service manifests to deploy the pods.

* Deploy the pods based on the yaml files to the respective ports.

### Test the Setup:

* Test the replication setup by writing data to the master server and verifying that it is replicated to the slave servers.

* Test the load balancing by sending traffic to ProxySQL and verifying that it is distributed across the MariaDB instances.

* Test the failover mechanism by simulating a failure of the master server and verifying that one of the slaves is automatically promoted to become the new master.

### Important Considerations:

* Persistent Storage: Ensure that the MariaDB data directory is stored on a persistent volume that survives container restarts. Use Kubernetes PersistentVolumeClaims (PVCs) to achieve this.

* Networking: Configure networking properly to allow MariaDB clients to connect to the database server.

* Security: Secure the K3s cluster, MariaDB instances, and ProxySQL.

* Monitoring: Implement monitoring to track the performance of the K3s cluster, MariaDB instances, and ProxySQL.

* Automated Failover: Implement an automated failover mechanism to promote a slave to master in case the main master node fails.

* Resource Allocation: Each server can run at most, a single master. Make sure that resources are not strained and the servers are all healthy

### Next Steps:

* Create the Container Images: Create Docker images for MariaDB and ProxySQL, or use existing images from Docker Hub.

* Write Kubernetes Manifests: Create the Kubernetes Deployment and Service manifests.

* Deploy to K3s: Deploy the manifests to your K3s cluster

* Do Testing: Always continue to check the ports to make sure that everything is running smoothly.

* Load to K3s: Then scale traffic to the K3s over time to ensure that the cluster can handle it.

## Pod : Podman pod 

`podman pod create --name pod-mariadbmaster -p 3306:3306 --network db-stack `

`podman run -d --pod pod-mariadbmaster --name mariadb-master -v ./db_master_data:/var/lib/mysql -v ./master.cnf:/etc/mysql/conf.d/mysql.cnf -e MARIADB_ROOT_PASSWORD=1 mariadb:latest`

`podman pod create --name pod-mariadbslave -p 3307:3306 --network db-stack `

`podman run -d --pod pod-mariadbslave --name mariadb-slave -v ./db_slave_data:/var/lib/mysql -v ./slave.cnf:/etc/mysql/conf.d/mysql.cnf -e MARIADB_ROOT_PASSWORD=1 mariadb:latest`

`podman pod create --name pod-mariadbslave2 -p 3308:3306 --network db-stack `

`podman run -d --pod pod-mariadbslave2 --name mariadb-slave2 -v ./db_slave2_data:/var/lib/mysql -v ./slave2.cnf:/etc/mysql/conf.d/mysql.cnf -e MARIADB_ROOT_PASSWORD=1 mariadb:latest`

## Sample Instance : Database and Table for Master Slave
` CREATE DATABASE football;`
` USE football;`
` CREATE TABLE players (name varchar(50) DEFAULT NULL,position varchar(50) DEFAULT NULL);`
` INSERT INTO players VALUES ('Lionel Messi','Forward');`
## Create user : replication user for Master Slave 
##### Note: replication user is exclusive only for master instance
` GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' IDENTIFIED BY '123'; #enter password`
` FLUSH PRIVILEGES;`
` show master status;`

## Set up :  Do this for all the rest of salve database  instance

`STOP SLAVE;
RESET SLAVE ALL;
 CHANGE MASTER TO 
   MASTER_HOST='10.89.0.6', -- local ip addres  of master database 
   MASTER_USER='slave_user', -- previous user create 
   MASTER_PASSWORD='123',
   MASTER_LOG_FILE='master-bin.000004', -- this will appear in the  master database using Show master staus
   master_use_gtid=slave_pos, -- set the  GTID in slave_POS
   MASTER_LOG_POS=343; -- same goes here `
`START SLAVE;`
`show slave status \G;`

### Create user : master and slave 
`CREATE USER 'maxscale'@'%' IDENTIFIED BY 'maxscale';`
`GRANT SUPER, REPLICA MONITOR, REPLICATION CLIENT, REPLICATION SLAVE, SHOW DATABASES, EVENT, PROCESS, SLAVE MONITOR, READ_ONLY ADMIN ON *.* TO 'maxscale'@'%';`
`GRANT SELECT ON mysql.* TO 'maxscale'@'%';`
`FLUSH PRIVILEGES;`

### Container engine : Podman for this example!
`podman pod create --name pod-proxysql -p 6033:6033 -p 6032:6032 -p 6080:6080 --network db-stack `

`podman run -d --pod pod-proxysql --name proxysql -v proxysql_data:/var/lib/proxysql proxysql/proxysql`

### Access: Enter podman shell state log in
`podman exec -it proxysql sh`

`mysql -u admin -padmin -h 127.0.0.1 -P 6032`
### Setup : Copy the following

`INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight) 
VALUES (10, '10.89.0.6', 3306, 100), 
       (20, '10.89.0.6', 3306, 100), 
       (40, '10.89.0.7', 3306, 100), 
       (30, '10.89.0.7', 3306, 100), 
       (30, '10.89.0.8', 3306, 100);`

`LOAD MYSQL SERVERS TO RUNTIME;`

`SAVE MYSQL SERVERS TO DISK;`

# 

`INSERT INTO mysql_query_rules (rule_id, active, match_digest, destination_hostgroup, apply) VALUES 
(1, 1, "^SELECT.*", 20, 1),
(2, 1, ".*", 10, 1);`

`LOAD MYSQL QUERY RULES TO RUNTIME;`
`SAVE MYSQL QUERY RULES TO DISK;`

#

`INSERT INTO mysql_users (username, password, default_hostgroup) VALUES 
('maxscale', 'maxscale', 10);`

`LOAD MYSQL USERS TO RUNTIME;`
`SAVE MYSQL USERS TO DISK;`

#

`UPDATE global_variables SET variable_value='maxscale' WHERE variable_name='mysql-monitor_username';`
`UPDATE global_variables SET variable_value='maxscale' WHERE variable_name='mysql-monitor_password';`

`LOAD MYSQL VARIABLES TO RUNTIME;`
`SAVE MYSQL VARIABLES TO DISK;`

#

`INSERT INTO mysql_group_replication_hostgroups (writer_hostgroup, backup_writer_hostgroup, reader_hostgroup, offline_hostgroup, active, max_writers, writer_is_also_reader, max_transactions_behind) VALUES (20, 40, 30, 10, 1, 2, 1, 100);`

`LOAD MYSQL SERVERS TO RUNTIME;`
`SAVE MYSQL SERVERS TO DISK;`

#

# Debugging
`podman logs --tail 1000 mariadb-master`

podman pull docker.io/library/mariadb:latest
podman pull docker.io/proxysql/proxysql

relay_master_log_file:master-bin.000005 
exec_master_log_pos_343

SELECT BINLOG_GTID_POS('master-bin.000002', '1930');

SET GLOBAL gtid_slave_pos='0-1-9';

SHOW VARIABLES LIKE 'log_bin';

DROP USER 'maxscale'@'%';

SELECT user, host FROM mysql.user;

SHOW GRANTS FOR 'slave_user'@'%';


SELECT @@gtid_current_pos;
SELECT @@gtid_slave_pos;

SELECT variable_name, variable_value 
FROM global_variables 
WHERE variable_name LIKE 'mysql-monitor_%';

podman inspect mariadb-slave | grep "IPAddress"

podman stop $(podman ps -q)
podman rm $(podman ps -aq)
podman volume prune -f
podman network prune -f

