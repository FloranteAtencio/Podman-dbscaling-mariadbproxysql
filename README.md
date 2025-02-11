# Podman-dbscaling-mariadbproxysql


CREATE DATABASE football;\
 USE football;\`
 CREATE TABLE players (name varchar(50) DEFAULT NULL,position varchar(50) DEFAULT NULL);\
 INSERT INTO players VALUES ('Lionel Messi','Forward');\
 GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' IDENTIFIED BY '123'; #enter password\
 FLUSH PRIVILEGES;\
 show master status;

STOP SLAVE; \
RESET SLAVE ALL; \
 CHANGE MASTER TO \
   MASTER_HOST='mariadb-master',\
   MASTER_USER='slave_user',\
   MASTER_PASSWORD='123',\
   MASTER_LOG_FILE='master-bin.000002',\
   master_use_gtid=slave_pos,\
   MASTER_LOG_POS=1199;\
START SLAVE;\
SHOW SLAVE STATUS \G;


for  master and slave
CREATE USER 'maxscale'@'%' IDENTIFIED BY 'maxscale';\
GRANT SUPER, REPLICA MONITOR, REPLICATION CLIENT, REPLICATION SLAVE, SHOW DATABASES, EVENT, PROCESS, SLAVE MONITOR, READ_ONLY ADMIN ON *.* TO 'maxscale'@'%';\
GRANT SELECT ON mysql.* TO 'maxscale'@'%';\
FLUSH PRIVILEGES;\


podman run -d --name proxysql \
    --network db-stack \
    -p 6032:6032 -p 6033:6033 -p 6080:6080 \
    -v proxysql_data:/var/lib/proxysql \
    proxysql/proxysql

podman exec -it proxysql sh
mysql -u admin -padmin -h 127.0.0.1 -P 6032

INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight) VALUES \
(10, 'mariadb-master', 3306, 100), \
(20, 'mariadb-slave', 3306, 100),\
(20, 'mariadb-slave2', 3306, 100);\

LOAD MYSQL SERVERS TO RUNTIME;\
SAVE MYSQL SERVERS TO DISK;\

`INSERT INTO mysql_query_rules (rule_id, active, match_digest, destination_hostgroup, apply) VALUES \
(1, 1, "^SELECT.*", 20, 1),\
(2, 1, ".*", 10, 1);\`

LOAD MYSQL QUERY RULES TO RUNTIME;\
SAVE MYSQL QUERY RULES TO DISK;\

INSERT INTO mysql_users (username, password, default_hostgroup) VALUES \
('maxscale', 'maxscale', 10);\

UPDATE mysql_users SET password = 'maxscale' WHERE username = 'maxscale';\

LOAD MYSQL USERS TO RUNTIME;\
SAVE MYSQL USERS TO DISK;\

UPDATE global_variables SET variable_value='maxscale' WHERE variable_name='mysql-monitor_username';\
UPDATE global_variables SET variable_value='maxscale' WHERE variable_name='mysql-monitor_password';\

LOAD MYSQL VARIABLES TO RUNTIME;\
SAVE MYSQL VARIABLES TO DISK;\

INSERT INTO mysql_replication_hostgroups (writer_hostgroup, reader_hostgroup) VALUES (10, 20);
LOAD MYSQL SERVERS TO RUNTIME; \
SAVE MYSQL SERVERS TO DISK;\

podman logs --tail 1000 mariadb-master\

podman pull docker.io/library/mariadb:latest\
podman pull docker.io/proxysql/proxysql\

relay_master_log_file:master-bin.000005 \
exec_master_log_pos_343\

SELECT BINLOG_GTID_POS('master-bin.000002', '1930');

SET GLOBAL gtid_slave_pos='0-1-9';

SHOW VARIABLES LIKE 'log_bin';

DROP USER 'maxscale'@'%';

SELECT user, host FROM mysql.user;

SHOW GRANTS FOR 'slave_user'@'%';

SELECT @@gtid_current_pos;
SELECT @@gtid_slave_pos;

podman stop $(podman ps -q)

podman rm $(podman ps -aq)

podman volume prune -f

podman network prune -f


SELECT variable_name, variable_value 
FROM global_variables 
WHERE variable_name LIKE 'mysql-monitor_%';



