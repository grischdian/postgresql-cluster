# PostgreSQL Cluster with pgpool-II on ** RHEL 8 **
#### Mainly [this](https://www.pgpool.net/docs/42/en/html/example-cluster.html) tutorial was used here.
#### Info about [streaming replication](https://wiki.postgresql.org/wiki/Streaming_Replication)
#### pgpool-II [Docu](https://www.pgpool.net/docs/42/en/html/index.html)
### This covers a postgres-cluster with one primary Server and pgpool-II sperated from them.
We assue that you create only one User which is able to check and do replication and serving the application

We assume to use this machines but you can Scale as much as you like:

| Servername | Role | 
| -------- | -------- | 
| pgpool-1     | pgpool     |
| pgpool-2     | pgpool     |
| pg1     | postgres - primary     | 

Installation PostgreSQL

- Install postgresql-XX-server package on `pg1`
** Next step for RHEL 8 only **
- Since postgresql on rhel 8 changed the directory structure and the pgpool is using the coorect way we have to symlink some directories first
```
mkdir -p /usr/pgsql-10/share
cd /usr/pgsql-10
ln -s /usr/lib64/pgsql lib
cd share
ln -s /usr/share/pgsql/extension .

# Qutickfix for 
mkdir -p /usr/pgsql-10/share
cd /usr/pgsql-10
ln -s /opt/rh/rh-postgresql10/root/usr/lib64/pgsql lib
cd share
ln -s /opt/rh/rh-postgresql10/root/usr/share/pgsql/extension .
```
- Install pgpool-II-pg10-extensions on `pg1`
    - yes this will install pgpool as well as a dependecy
    - we don't start pgpool later - we just need the extension
- Initialize Database on ```pg1```
- Create /var/lib/pgsql/archivedir on ```pg1```
- Configure Database according your needs
    - log settings / timezone etc.
- Ensure the following parameters are set: (The two comments need to be removed later)
```
listen_addresses = '*'
archive_mode = on
archive_command = 'cp "%p" "/var/lib/pgsql/archivedir/%f"'
#synchronous_commit = remote_apply
#synchronous_standby_names = '*'
max_wal_senders = 10
max_replication_slots = 10
wal_level = replica
hot_standby = on
wal_log_hints = on
```
- Create 1 User:
    - owncloud - for app and replication

```
psql -U postgres -p 5432
postgres=# SET password_encryption = 'md5';
postgres=# CREATE ROLE owncloud WITH REPLICATION LOGIN;
postgres=# \password owncloud
postgres=# GRANT pg_monitor TO owncloud;
postgres=# CREATE DATABASE owncloudDB OWNER owncloud;
```
- Create Extenstion
```
psql template1 -c "CREATE EXTENSION pgpool_recovery"
```

- Create ```pg_hba.conf```
```
host    all             all             samenet                 md5

```
Start postgres on `pg1`

DONE

You will find pgpool at the end of this file

### This covers a postgres-cluster with multiple Server and pgpool-II seperated from them.
### **ONLY for RHEL 7** 
You have to modify on `pg*` Path and Content
- `/var/lib/pgsql/10/data/recovery_1st_stage`
	- `PGHOME=/usr/pgsql-10`
- `/var/lib/pgsql/10/data/pgpool_remote_start`
	- `PGHOME=/usr/pgsql-10`

We assume to use this machines but you can Scale as much as you like:

| Servername | Role | 
| -------- | -------- | 
| pgpool-1     | pgpool     |
| pg1     | postgres - primary     | 
| pg2     | postgres - standby     |
| pgX     | postgres - standby     |

Installation PostgreSQL

- Install postgresql-XX-server package on all `pg*`
** Next step for RHEL 8 only **
- Since postgresql on rhel 8 changed the directory structure and the pgpool is using the coorect way we have to symlink some directories first
```
mkdir -p /usr/pgsql-10/share
cd /usr/pgsql-10
ln -s /usr/lib64/pgsql lib
cd share
ln -s /usr/share/pgsql/extension .
```
- Install pgpool-II-pg10-extensions on all `pg*`
    - yes this will install pgpool as well as a dependecy
    - we don't start pgpool later - we just need the extension
- Initialize Database on ```pg1```
- Create /var/lib/pgsql/archivedir on ```pg*```
- Configure Database according your needs
    - log settings / timezone etc.
- Ensure the following parameters are set: (The two comments need to be removed later)
```
listen_addresses = '*'
archive_mode = on
archive_command = 'cp "%p" "/var/lib/pgsql/archivedir/%f"'
#synchronous_commit = remote_apply
#synchronous_standby_names = '*'
max_wal_senders = 10
max_replication_slots = 10
wal_level = replica
hot_standby = on
wal_log_hints = on
```
- Create 3 User:
    - repl - for replication
    - pgpool - for healthcheck and replication delay check
    - postgres - for online recovery

```
psql -U postgres -p 5432
postgres=# SET password_encryption = 'md5';
postgres=# CREATE ROLE pgpool WITH LOGIN;
postgres=# CREATE ROLE repl WITH REPLICATION LOGIN;
postgres=# \password pgpool
postgres=# \password repl
postgres=# \password postgres
postgres=# GRANT pg_monitor TO pgpool;
```
- Create Extenstion
```
psql template1 -c "CREATE EXTENSION pgpool_recovery"
```

- Create ```pg_hba.conf```
```
host    all             all             samenet                 md5
host    replication     all             samenet                 md5

```
- To use the automated failover and online recovery of Pgpool-II, the settings that allow passwordless SSH to all backend servers between Pgpool-II execution user (default postgres user) and postgres user are necessary.

- Create ssh-key for user `postgres` on `pg1`
```
su - postgres
ssh-keygen -t rsa -f id_rsa_pgpool
cd ~/.ssh
cat id_rsa_pgpool >> authorized_keys
```
- Copy private key and authorized_keys all `pg*` in Postgres `~/.ssh`
- Please ensure that passwordless login from all `pg*` to all `pg*` is possible. e.g
    - ```[pg1]# ssh postgres@pg1```
    - ```[pg1]# ssh postgres@pg2```
    - ```[pg1]# ssh postgres@pgX```
    - ```[pg2]# ssh postgres@pg1```
    - ```[pg2]# ssh postgres@pg2```
    - ```[pg2]# ssh postgres@pgX```
    - ...
        
- To allow repl user without specifying password for streaming replication and online recovery, and execute pg_rewind using postgres, we create the .pgpass file in postgres user's home directory and change the permission to 600 on each pg*`

** Please make sure to put the hostname of each server in the first column and ensure that DNS is configured properly**
```
su - postgres
vi /var/lib/pgsql/.pgpass
pg1:5432:replication:repl:<repl user password>
pg2:5432:replication:repl:<repl user passowrd>
pgX:5432:replication:repl:<repl user passowrd>
pg1:5432:postgres:postgres:<postgres user passowrd>
pg2:5432:postgres:postgres:<postgres user passowrd>
pgX:5432:postgres:postgres:<postgres user passowrd>
chmod 600  /var/lib/pgsql/.pgpass
```

- create Scripts with the following content:
    - ensure owner is `postgres`
    - ensure group is `postgres`
    - ensure mode is `700`

/var/lib/pgsql/10/data/recovery_1st_stage

```
#!/bin/bash
# This script is executed by "recovery_1st_stage" to recovery a Standby node.

set -o xtrace

PRIMARY_NODE_PGDATA="$1"
DEST_NODE_HOST="$2"
DEST_NODE_PGDATA="$3"
PRIMARY_NODE_PORT="$4"
DEST_NODE_ID="$5"
DEST_NODE_PORT="$6"

PRIMARY_NODE_HOST=$(hostname)
PGHOME=/usr
ARCHIVEDIR=/var/lib/pgsql/archivedir
REPLUSER=repl

echo recovery_1st_stage: start: pg_basebackup for Standby node $DEST_NODE_ID

## Test passwrodless SSH
ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@${DEST_NODE_HOST} -i ~/.ssh/id_rsa_pgpool ls /tmp > /dev/null

if [ $? -ne 0 ]; then
    echo recovery_1st_stage: passwrodless SSH to postgres@${DEST_NODE_HOST} failed. Please setup passwrodless SSH.
    exit 1
fi

## Get PostgreSQL major version
PGVERSION=`${PGHOME}/bin/initdb -V | awk '{print $3}' | sed 's/\..*//' | sed 's/\([0-9]*\)[a-zA-Z].*/\1/'`
if [ $PGVERSION -ge 12 ]; then
    RECOVERYCONF=${DEST_NODE_PGDATA}/myrecovery.conf
else
    RECOVERYCONF=${DEST_NODE_PGDATA}/recovery.conf
fi

## Create replication slot "${DEST_NODE_HOST}"
${PGHOME}/bin/psql -p ${PRIMARY_NODE_PORT} << EOQ
SELECT pg_create_physical_replication_slot('${DEST_NODE_HOST}');
EOQ

## Execute pg_basebackup to recovery Standby node
ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@$DEST_NODE_HOST -i ~/.ssh/id_rsa_pgpool "

    set -o errexit

    rm -rf $DEST_NODE_PGDATA
    rm -rf $ARCHIVEDIR/*

    ${PGHOME}/bin/pg_basebackup -h $PRIMARY_NODE_HOST -U $REPLUSER -p $PRIMARY_NODE_PORT -D $DEST_NODE_PGDATA -X stream

    if [ ${PGVERSION} -ge 12 ]; then
        sed -i -e \"\\\$ainclude_if_exists = '$(echo ${RECOVERYCONF} | sed -e 's/\//\\\//g')'\" \
               -e \"/^include_if_exists = '$(echo ${RECOVERYCONF} | sed -e 's/\//\\\//g')'/d\" ${DEST_NODE_PGDATA}/postgresql.conf
    fi

    cat > ${RECOVERYCONF} << EOT
primary_conninfo = 'host=${PRIMARY_NODE_HOST} port=${PRIMARY_NODE_PORT} user=${REPLUSER} application_name=${DEST_NODE_HOST} passfile=''/var/lib/pgsql/.pgpass'''
recovery_target_timeline = 'latest'
restore_command = 'scp ${PRIMARY_NODE_HOST}:${ARCHIVEDIR}/%f %p'
primary_slot_name = '${DEST_NODE_HOST}'
EOT

    if [ ${PGVERSION} -ge 12 ]; then
        touch ${DEST_NODE_PGDATA}/standby.signal
    else
        echo \"standby_mode = 'on'\" >> ${RECOVERYCONF}
    fi

    sed -i \"s/#*port = .*/port = ${DEST_NODE_PORT}/\" ${DEST_NODE_PGDATA}/postgresql.conf
"

if [ $? -ne 0 ]; then

    ${PGHOME}/bin/psql -p ${PRIMARY_NODE_PORT} << EOQ
SELECT pg_drop_replication_slot('${DEST_NODE_HOST}');
EOQ

    echo recovery_1st_stage: end: pg_basebackup failed. online recovery failed
    exit 1
fi

echo recovery_1st_stage: end: recovery_1st_stage is completed successfully
exit 0

```

/var/lib/pgsql/10/data/pgpool_remote_start
```
#!/bin/bash
# This script is run after recovery_1st_stage to start Standby node.

set -o xtrace

DEST_NODE_HOST="$1"
DEST_NODE_PGDATA="$2"

PGHOME=/usr

echo pgpool_remote_start: start: remote start Standby node $DEST_NODE_HOST

## Test passwrodless SSH
ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@${DEST_NODE_HOST} -i ~/.ssh/id_rsa_pgpool ls /tmp > /dev/null

if [ $? -ne 0 ]; then
    echo pgpool_remote_start: passwrodless SSH to postgres@${DEST_NODE_HOST} failed. Please setup passwrodless SSH.
    exit 1
fi

## Start Standby node
ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@$DEST_NODE_HOST -i ~/.ssh/id_rsa_pgpool "
    $PGHOME/bin/pg_ctl -l /dev/null -w -D $DEST_NODE_PGDATA start
"

if [ $? -ne 0 ]; then
    echo pgpool_remote_start: $DEST_NODE_HOST PostgreSQL start failed.
    exit 1
fi

echo pgpool_remote_start: end: PostgreSQL on $DEST_NODE_HOST is started successfully.
exit 0

```

- Remove the Comments in postgresql.conf
```
synchronous_standby_names = '*'        # standby servers that provide sync rep
synchronous_commit = remote_apply              # synchronization level;
```
    
- Start ONLY `pg1`


## Installation pgpool-II (not finished yep)
- Install pgpool-II-pg10
- Each node diffrent ID in `/etc/pgpool-II/pgpool_node_id` (Just a number)
- Use /etc/pgpool-II/pgpool.conf.sample-stream -> copy to pgpool.conf
- Change the following parameter:
```
listen_addresses = '*'
port = 5432
sr_check_user = 'pgpool'
sr_check_password = ''
health_check_period = 5
health_check_timeout = 30
health_check_user = 'pgpool'
health_check_password = ''
health_check_max_retries = 3

backend_hostname0 = 'server1'
backend_port0 = 5432
backend_weight0 = 1
backend_data_directory0 = '/var/lib/pgsql/10/data'
backend_flag0 = 'ALLOW_TO_FAILOVER'
backend_application_name0 = 'server1'


backend_hostname1 = 'server2'
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/var/lib/pgsql/10/data'
backend_flag1 = 'ALLOW_TO_FAILOVER'
backend_application_name1 = 'server2'

enable_pool_hba = on
pool_passwd = 'pool_passwd'

failover_command = '/etc/pgpool-II/failover.sh %d %h %p %D %m %H %M %P %r %R %N %S'
follow_primary_command = '/etc/pgpool-II/follow_primary.sh %d %h %p %D %m %H %M %P %r %R'
recovery_user = 'postgres'
recovery_password = ''
recovery_1st_stage_command = 'recovery_1st_stage'


```

### Create pcpass File in postgres home

Create .pcppass File for PCP passwordless execution
```
su - postgres
echo 'localhost:9898:pgpool:<pgpool user password>' > ~/.pcppass
chmod 600 ~/.pcppass
```

### Create configfiles in /etc/pgpool-II

create `pool_passwd`
```
postgres:TEXT$PASSWORD
pgpool:TEXT$PASSWORD
ocdb:TEXT$PASSWORD
```
create pcp.conf
->`echo 'pgpool:'`pg_md5 PCP $PASSWORD_FOR_PGPOOL_USER` >> /etc/pgpool-II/pcp.conf`

Create .pcppass File for PCP passwordless execution
```
su - postgres
echo 'localhost:9898:pgpool:<pgpool user password>' > ~/.pcppass
chmod 600 ~/.pcppass
```

Create `pg_hba.conf` 
```
host    all         pgpool           0.0.0.0/0          md5
host    all         postgres         0.0.0.0/0          md5
host    all         $OCUSER          $NETWORK/24        md5
```
Create `failover.sh`
```
ssh -tt pgpool@$6 sudo -u postgres /var/lib/pgsql/data/failover.sh $@
```
Create `follow_primary.sh`
```
ssh -tt pgpool@$6 sudo -u postgres /var/lib/pgsql/data/follow_primary.sh $@
```

Start pgpool-II and let all Server join the Cluster
```
pcp_recovery_node -h pgpool -p 9898 -U pgpool -n 0 -v -d
```
Check added Nodes and Cluster status
```
psql -h localhost -p 5432 -U pgpool postgres -c "show pool_nodes"
```
