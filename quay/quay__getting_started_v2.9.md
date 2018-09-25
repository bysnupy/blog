# QUAY Basic Installation Practice

## Description 
This installation practice is for testing and verification of the installation processes.

## Environment

Component | Value 
-|-
RHEL | 7.5
PostgreSQL | 9.6
Redis | 3.2


## Installation Steps

This steps are required `RHEL` minimal installation.

### Register required subscription and Update the packages

~~~
# subscription-manager register --username=<USERNAME>
<password>

# subscription-manager repos --disable="*"

# subscription-manager repos --enable="rhel-7-server-rpms" \
                             --enable="rhel-7-server-extras-rpms"
                             
# yum update -y
~~~

### Set up `Docker` service

~~~
# yum install docker

# systemctl enable docker.service

# systemctl start docker.service

# systemctl status docker.service
~~~

### Set up Authentication to `Quay.io` for pulling `QUAY` container image.

If you authenticate the certified credential to `Quay.io`, then the `/root/.docker/config.json` is generated. You can use the authentication key in the `json`.


### Deploy a Database (PostgreSQL)

~~~
# mkdir -p /mnt/pgdata

# chmod 777 /mnt/pgdata

# export POSTGRESQL_DATABASE=quay_db
# export POSTGRESQL_PASSWORD=quay_db_pass
# export POSTGRESQL_USER=quay_db_user
# export POSTGRESQL_ADMIN_PASSWORD=postgres
# export POSTGRESQL_CONTAINER_NAME=postgres

# docker run --detach --restart=always --env POSTGRESQL_DATABASE=${POSTGRESQL_DATABASE} \
  --env POSTGRESQL_USER=${POSTGRESQL_USER} --env POSTGRESQL_PASSWORD=${POSTGRESQL_PASSWORD} \
  --env POSTGRESQL_ADMIN_PASSWORD=${POSTGRESQL_ADMIN_PASSWORD} --name ${POSTGRESQL_CONTAINER_NAME} --publish 5432:5432 \
  -v /mnt/pgdata:/var/lib/pgsql/data:Z registry.access.redhat.com/rhscl/postgresql-96-rhel7
~~~

Check the database connectivity using `psql` client.
~~~
# subscription-manager repos --enable="rhel-server-rhscl-7-rpms"
# yum install rh-postgresql96-postgresql -y
# firewall-cmd --permanent --add-port 5432/tcp
# firewall-cmd --reload
# LD_LIBRARY_PATH=/opt/rh/rh-postgresql96/root/usr/lib64 /opt/rh/rh-postgresql96/root/usr/bin/psql -U quay_db_user -h <HOST_IP_ADDRESS> quay_db
~~~

### Deploy `Redis`

~~~
# mkdir -p /mnt/redis

# chmod 777 /mnt/redis

# docker run -d --restart=always -p 6379:6379 \
  -v /mnt/redis:/var/lib/redis/data:Z registry.access.redhat.com/rhscl/redis-32-rhel7
~~~

### Deploy `QUAY`
~~~
# mkdir -p /var/run/quay/config
# mkdir -p /var/run/quay/storage
# docker run --restart=always -p 443:443 -p 80:80 \
     --privileged=true \
     -v /var/run/quay/config:/conf/stack \
     -v /var/run/quay/storage:/datastorage \
     -d quay.io/coreos/quay:v2.9.2
~~~
