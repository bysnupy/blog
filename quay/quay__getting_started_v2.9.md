# QUAY Installation v2.9

It's a summary of a QUAY installation. You can refer the details of the installation steps from [here](https://access.redhat.com/documentation/en-us/red_hat_quay/2.9/html-single/getting_started_with_red_hat_quay/index)

## Environments
Item|Values
-|-
Version| QUAY v2.9.1
Type|Docker container image based

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


