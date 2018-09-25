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

Grant SuperUser role to POSTGRESQL_USER for enabling additional extension installation.
~~~
# docker ps
CONTAINER ID        IMAGE                                                  COMMAND                  CREATED             STATUS              PORTS                                                NAMES
e779937e30c2        registry.access.redhat.com/rhscl/postgresql-96-rhel7   "container-entrypo..."   30 minutes ago      Up 30 minutes       0.0.0.0:5432->5432/tcp                               postgres

# docker exec -it e779937e30c2 /bin/bash

bash-4.2$ psql 

psql (9.6.10)
Type "help" for help.

postgres=# \du
                                     List of roles
  Role name   |                         Attributes                         | Member of 
--------------+------------------------------------------------------------+-----------
 postgres     | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 quay_db_user |                                                            | {}
 
postgres=# ALTER ROLE quay_db_user WITH SUPERUSER;
ALTER ROLE
postgres=# \du
                                     List of roles
  Role name   |                         Attributes                         | Member of 
--------------+------------------------------------------------------------+-----------
 postgres     | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 quay_db_user | Superuser                                                  | {}

postgres=# \q
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

### Completing the Guided Setup

Open the `6379(redis)`, `80(http)` and `443(https)` port before setup.

~~~
# firewall-cmd --permanent --add-port 6379/tcp --add-service http --add-service https

# firewall-cmd --reload
~~~

Access `http://<HOSTNAME>/setup`

- Input Database connectivity information.
![Step1](https://github.com/bysnupy/blog/images/quay_guided_step1.png)

- Create Administrator account of `QUAY`.
![Step2](https://github.com/bysnupy/blog/images/quay_guided_step2.png)

- Configure additional setup in the `QUAY`.
![Step3](https://github.com/bysnupy/blog/images/quay_guided_step3.png)

- Restart `QUAY` container for taking effect.
![Step4](https://github.com/bysnupy/blog/images/quay_guided_step4.png)

- Complete the `QUAY` initialization.
![Step5](https://github.com/bysnupy/blog/images/quay_guided_step5.png)
