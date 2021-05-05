# PostgreSQL Cluster with pgpool-II on ** RHEL 8 **
### This covers a postgres-cluster with multiple Server and pgpool-II seperated from them.
### **ONLY for RHEL 7** 
You have to modify on `pg*` Path and Content
- `/var/lib/pgsql/10/data/recovery_1st_stage`
	- `PGHOME=/usr/pgsql-10`
- `/var/lib/pgsql/10/data/pgpool_remote_start`
	- `PGHOME=/usr/pgsql-10`
- `/var/lib/pgsql/10/data/failover.sh`
	- `PGHOME=/usr/pgsql-10`
- `/var/lib/pgsql/10/data/follow_primary.sh`
- sudo rules
and on `pgpool*`
- `/etc/pgpool-II/failover.sh`
	- `ssh -tt pgpool@$6 sudo -u postgres /var/lib/pgsql/data/failover.sh $@

- `/etc/pgpool-II/follow_primary.sh`
	- `ssh -tt pgpool@$6 sudo -u postgres /var/lib/pgsql/data/follow_primary.sh $@`

#### Mainly [this](https://www.pgpool.net/docs/42/en/html/example-cluster.html) tutorial was used here.
#### Info about [streaming replication](https://wiki.postgresql.org/wiki/Streaming_Replication)
#### pgpool-II [Docu](https://www.pgpool.net/docs/42/en/html/index.html)

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
/var/lib/pgsql/10/data/failover.sh

```
#!/bin/bash
# This script is run by failover_command.

set -o xtrace

# Special values:
# 1)  %d = failed node id
# 2)  %h = failed node hostname
# 3)  %p = failed node port number
# 4)  %D = failed node database cluster path
# 5)  %m = new main node id
# 6)  %H = new main node hostname
# 7)  %M = old main node id
# 8)  %P = old primary node id
# 9)  %r = new main port number
# 10) %R = new main database cluster path
# 11) %N = old primary node hostname
# 12) %S = old primary node port number
# 13) %% = '%' character

FAILED_NODE_ID="$1"
FAILED_NODE_HOST="$2"
FAILED_NODE_PORT="$3"
FAILED_NODE_PGDATA="$4"
NEW_MAIN_NODE_ID="$5"
NEW_MAIN_NODE_HOST="$6"
OLD_MAIN_NODE_ID="$7"
OLD_PRIMARY_NODE_ID="$8"
NEW_MAIN_NODE_PORT="$9"
NEW_MAIN_NODE_PGDATA="${10}"
OLD_PRIMARY_NODE_HOST="${11}"
OLD_PRIMARY_NODE_PORT="${12}"

PGHOME=/usr


echo failover.sh: start: failed_node_id=$FAILED_NODE_ID old_primary_node_id=$OLD_PRIMARY_NODE_ID failed_host=$FAILED_NODE_HOST new_main_host=$NEW_MAIN_NODE_HOST

## If there's no main node anymore, skip failover.
if [ $NEW_MAIN_NODE_ID -lt 0 ]; then
    echo failover.sh: All nodes are down. Skipping failover.
	exit 0
fi

## Test passwrodless SSH
ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@${NEW_MAIN_NODE_HOST} -i ~/.ssh/id_rsa_pgpool ls /tmp > /dev/null

if [ $? -ne 0 ]; then
    echo failover.sh: passwrodless SSH to postgres@${NEW_MAIN_NODE_HOST} failed. Please setup passwrodless SSH.
    exit 1
fi

## If Standby node is down, skip failover.
if [ $FAILED_NODE_ID -ne $OLD_PRIMARY_NODE_ID ]; then

    ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@$OLD_PRIMARY_NODE_HOST -i ~/.ssh/id_rsa_pgpool "
        ${PGHOME}/bin/psql -p $OLD_PRIMARY_NODE_PORT -c \"SELECT pg_drop_replication_slot('${FAILED_NODE_HOST}')\"
    "

    if [ $? -ne 0 ]; then
        echo failover.sh: Standby node is down. Skipping failover.
        echo failover.sh: end: drop replication slot "${FAILED_NODE_HOST}" failed
        exit 1
    fi

    echo failover.sh: end: Standby node is down. Skipping failover.
    exit 0
fi

## Promote Standby node.
echo failover.sh: Primary node is down, promote standby node ${NEW_MAIN_NODE_HOST}.

ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
    postgres@${NEW_MAIN_NODE_HOST} -i ~/.ssh/id_rsa_pgpool ${PGHOME}/bin/pg_ctl -D ${NEW_MAIN_NODE_PGDATA} -w promote

if [ $? -ne 0 ]; then
    echo failover.sh: end: failover failed
    exit 1
fi

echo failover.sh: end: new_main_node_id=$NEW_MAIN_NODE_ID is promoted to a primary
exit 0

```
/var/lib/pgsql/10/data/follow_primary.sh
```
#!/bin/bash
# This script is run after failover_command to synchronize the Standby with the new Primary.
# First try pg_rewind. If pg_rewind failed, use pg_basebackup.

set -o xtrace

# Special values:
# 1)  %d = node id
# 2)  %h = hostname
# 3)  %p = port number
# 4)  %D = node database cluster path
# 5)  %m = new primary node id
# 6)  %H = new primary node hostname
# 7)  %M = old main node id
# 8)  %P = old primary node id
# 9)  %r = new primary port number
# 10) %R = new primary database cluster path
# 11) %N = old primary node hostname
# 12) %S = old primary node port number
# 13) %% = '%' character

NODE_ID="$1"
NODE_HOST="$2"
NODE_PORT="$3"
NODE_PGDATA="$4"
NEW_PRIMARY_NODE_ID="$5"
NEW_PRIMARY_NODE_HOST="$6"
OLD_MAIN_NODE_ID="$7"
OLD_PRIMARY_NODE_ID="$8"
NEW_PRIMARY_NODE_PORT="$9"
NEW_PRIMARY_NODE_PGDATA="${10}"

PGHOME=/usr
ARCHIVEDIR=/var/lib/pgsql/archivedir
REPLUSER=repl
PCP_USER=pgpool
PGPOOL_PATH=/usr/bin
PCP_PORT=9898

echo follow_primary.sh: start: Standby node ${NODE_ID}

## Test passwrodless SSH
ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@${NEW_PRIMARY_NODE_HOST} -i ~/.ssh/id_rsa_pgpool ls /tmp > /dev/null

if [ $? -ne 0 ]; then
    echo follow_main.sh: passwrodless SSH to postgres@${NEW_PRIMARY_NODE_HOST} failed. Please setup passwrodless SSH.
    exit 1
fi

## Get PostgreSQL major version
PGVERSION=`${PGHOME}/bin/initdb -V | awk '{print $3}' | sed 's/\..*//' | sed 's/\([0-9]*\)[a-zA-Z].*/\1/'`

if [ $PGVERSION -ge 12 ]; then
    RECOVERYCONF=${NODE_PGDATA}/myrecovery.conf
else
    RECOVERYCONF=${NODE_PGDATA}/recovery.conf
fi

## Check the status of Standby
ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
postgres@${NODE_HOST} -i ~/.ssh/id_rsa_pgpool ${PGHOME}/bin/pg_ctl -w -D ${NODE_PGDATA} status


## If Standby is running, synchronize it with the new Primary.
if [ $? -eq 0 ]; then

    echo follow_primary.sh: pg_rewind for node ${NODE_ID}

    # Create replication slot "${NODE_HOST}"
    ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@${NEW_PRIMARY_NODE_HOST} -i ~/.ssh/id_rsa_pgpool "
        ${PGHOME}/bin/psql -p ${NEW_PRIMARY_NODE_PORT} -c \"SELECT pg_create_physical_replication_slot('${NODE_HOST}');\"
    "

    ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@${NODE_HOST} -i ~/.ssh/id_rsa_pgpool "

        set -o errexit

        ${PGHOME}/bin/pg_ctl -w -m f -D ${NODE_PGDATA} stop

        ${PGHOME}/bin/pg_rewind -D ${NODE_PGDATA} --source-server=\"user=postgres host=${NEW_PRIMARY_NODE_HOST} port=${NEW_PRIMARY_NODE_PORT}\"

        cat > ${RECOVERYCONF} << EOT
primary_conninfo = 'host=${NEW_PRIMARY_NODE_HOST} port=${NEW_PRIMARY_NODE_PORT} user=${REPLUSER} application_name=${NODE_HOST} passfile=''/var/lib/pgsql/.pgpass'''
recovery_target_timeline = 'latest'
restore_command = 'scp ${NEW_PRIMARY_NODE_HOST}:${ARCHIVEDIR}/%f %p'
primary_slot_name = '${NODE_HOST}'
EOT

        if [ ${PGVERSION} -ge 12 ]; then
            touch ${NODE_PGDATA}/standby.signal
        else
            echo \"standby_mode = 'on'\" >> ${RECOVERYCONF}
        fi

        ${PGHOME}/bin/pg_ctl -l /dev/null -w -D ${NODE_PGDATA} start

    "

    if [ $? -ne 0 ]; then
        echo follow_primary.sh: end: pg_rewind failed. Try pg_basebackup.

        ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@${NODE_HOST} -i ~/.ssh/id_rsa_pgpool "

            set -o errexit

            # Execute pg_basebackup
            rm -rf ${NODE_PGDATA}
            rm -rf ${ARCHIVEDIR}/*
            ${PGHOME}/bin/pg_basebackup -h ${NEW_PRIMARY_NODE_HOST} -U $REPLUSER -p ${NEW_PRIMARY_NODE_PORT} -D ${NODE_PGDATA} -X stream

            if [ ${PGVERSION} -ge 12 ]; then
                sed -i -e \"\\\$ainclude_if_exists = '$(echo ${RECOVERYCONF} | sed -e 's/\//\\\//g')'\" \
                       -e \"/^include_if_exists = '$(echo ${RECOVERYCONF} | sed -e 's/\//\\\//g')'/d\" ${NODE_PGDATA}/postgresql.conf
            fi

            cat > ${RECOVERYCONF} << EOT
primary_conninfo = 'host=${NEW_PRIMARY_NODE_HOST} port=${NEW_PRIMARY_NODE_PORT} user=${REPLUSER} application_name=${NODE_HOST} passfile=''/var/lib/pgsql/.pgpass'''
recovery_target_timeline = 'latest'
restore_command = 'scp ${NEW_PRIMARY_NODE_HOST}:${ARCHIVEDIR}/%f %p'
primary_slot_name = '${NODE_HOST}'
EOT

            if [ ${PGVERSION} -ge 12 ]; then
                touch ${NODE_PGDATA}/standby.signal
            else
                echo \"standby_mode = 'on'\" >> ${RECOVERYCONF}
            fi
        "

        if [ $? -ne 0 ]; then
            # drop replication slot
            ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@${NEW_PRIMARY_NODE_HOST} -i ~/.ssh/id_rsa_pgpool "
                ${PGHOME}/bin/psql -p ${NEW_PRIMARY_NODE_PORT} -c \"SELECT pg_drop_replication_slot('${NODE_HOST}')\"
            "

            echo follow_primary.sh: end: pg_basebackup failed
            exit 1
        fi

        # start Standby node on ${NODE_HOST}
        ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
                postgres@${NODE_HOST} -i ~/.ssh/id_rsa_pgpool $PGHOME/bin/pg_ctl -l /dev/null -w -D ${NODE_PGDATA} start

    fi

    # If start Standby successfully, attach this node
    if [ $? -eq 0 ]; then

        # Run pcp_attact_node to attach Standby node to Pgpool-II.
        ${PGPOOL_PATH}/pcp_attach_node -w -h localhost -U $PCP_USER -p ${PCP_PORT} -n ${NODE_ID}

        if [ $? -ne 0 ]; then
                echo follow_primary.sh: end: pcp_attach_node failed
                exit 1
        fi

    # If start Standby failed, drop replication slot "${NODE_HOST}"
    else

        ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@${NEW_PRIMARY_NODE_HOST} -i ~/.ssh/id_rsa_pgpool \
        ${PGHOME}/bin/psql -p ${NEW_PRIMARY_NODE_PORT} -c "SELECT pg_drop_replication_slot('${NODE_HOST}')"

        echo follow_primary.sh: end: follow primary command failed
        exit 1
    fi

else
    echo follow_primary.sh: failed_nod_id=${NODE_ID} is not running. skipping follow primary command
    exit 0
fi

echo follow_primary.sh: end: follow primary command is completed successfully
exit 0
```
- Ensure pgpool servers are able to login to unprivileged account e.g `pgpool-user` on `pg*`
    - add public key to authorized_keys 
- Ensure that `pgpool-user` is able to run `failover.sh` and `follow_primary.sh` from below via sudo without password
    - pgpool   ALL=(postgres) NOPASSWD: /var/lib/pgsql/data/failover.sh
    - pgpool   ALL=(postgres) NOPASSWD: /var/lib/pgsql/data/follow_primary.sh
    

- Remove the Comments in postgresql.conf
```
synchronous_standby_names = '*'        # standby servers that provide sync rep
synchronous_commit = remote_apply              # synchronization level;
```
    
- Start ONLY `pg1`


## Installation pgpool-II
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
