# QUAY Installation v2.9

It's a summary of a QUAY installation. You can refer the details of the installation steps from [here](https://access.redhat.com/documentation/en-us/red_hat_quay/2.9/html-single/getting_started_with_red_hat_quay/index)

## Environments
Item|Values
-|-
Version| QUAY v2.9.1
Type| container image based installation

## Installation Steps

### Prepare the installation

First of all, you should register subscription of Red Hat and enable the following repositories.
~~~
# subscription-manager repos --disable="*"
# subscription-manager repos \
    --enable="rhel-7-server-rpms" \
    --enable="rhel-7-server-extras-rpms"
# yum update -y
~~~

Let's install a docker package.
~~~
# yum install docker -y
# systemctl enable docker.service && systemctl start docker.service
~~~

If you have Red Hat account and authorization for searching the solution articles, you can read the solution for quay.io credential without creating a additional CoreOS account. It saved as `~/.docker/config.json`.

And then install the database for QUAY.

~~~
--- Setting up the required values as environment variables for easy setting.
# mkdir -p /mnt/hostmysql
# chmod 0777 /mnt/hostmysql
# export MYSQL_CONTAINER_NAME=mysql
# export MYSQL_DATABASE=enterpriseregistrydb
# export MYSQL_PASSWORD=redhat
# export MYSQL_USER=quayuser
# export MYSQL_ROOT_PASSWORD=redhat

--- Run the mysql databasae for metadata storage as container based on above values.
# docker run \
    --detach \
    --restart=always \
    --env MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD} \
    --env MYSQL_USER=${MYSQL_USER} \
    --env MYSQL_PASSWORD=${MYSQL_PASSWORD} \
    --env MYSQL_DATABASE=${MYSQL_DATABASE} \
    --name ${MYSQL_CONTAINER_NAME} \
    --publish 3306:3306 \
    -v /mnt/hostmysql:/var/lib/mysql/data:Z \
    registry.access.redhat.com/rhscl/mysql-57-rhel7

--- Create the local directory and run the redis for KVS as a docker container.
# mkdir -p /mnt/hostredis
# chmod 777 /mnt/hostredis
# docker run -d --restart=always -p 6379:6379 \
    -v /mnt/hostredis:/var/lib/redis/data:Z \
    registry.access.redhat.com/rhscl/redis-32-rhel7

--- Check the docker status.
# docker ps
CONTAINER ID        IMAGE                                             COMMAND                  CREATED             STATUS              PORTS                                                NAMES
ba3794b9a54a        registry.access.redhat.com/rhscl/redis-32-rhel7   "container-entrypo..."   6 days ago          Up 6 minutes        0.0.0.0:6379->6379/tcp                               elated_varahamihira
4ca907a1814d        registry.access.redhat.com/rhscl/mysql-57-rhel7   "container-entrypo..."   6 days ago          Up 6 minutes        0.0.0.0:3306->3306/tcp                               mysql
~~~

### Deploy the QUAY

It's so simple to deploy the QUAY.

~~~
# mkdir -p /var/run/quay/config
# mkdir -p /var/run/quay/storage
# docker run --restart=always -p 443:443 -p 80:80 \
   --privileged=true \
   -v /var/run/quay/config:/conf/stack \
   -v /var/run/quay/storage:/datastorage \
   -d quay.io/coreos/quay:v2.9.1
~~~

That's all. Then I'll check the status of it.

~~~
CONTAINER ID        IMAGE                                             COMMAND                  CREATED             STATUS              PORTS                                                NAMES
c4218e5d629f        quay.io/coreos/quay:v2.9.1                        "/bin/sh -c ./quay..."   6 days ago          Up 6 minutes        0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 8443/tcp   cocky_saha
ba3794b9a54a        registry.access.redhat.com/rhscl/redis-32-rhel7   "container-entrypo..."   6 days ago          Up 6 minutes        0.0.0.0:6379->6379/tcp                               elated_varahamihira
4ca907a1814d        registry.access.redhat.com/rhscl/mysql-57-rhel7   "container-entrypo..."   6 days ago          Up 6 minutes        0.0.0.0:3306->3306/tcp                               mysql
~~~

:information_source: You need the access through the browser for initialization database to adjust the firewall rules. But I just stop the firewalld.service for simple testing.

~~~
systemctl stop firewalld.service && systemctl disable firewalld.service
~~~
