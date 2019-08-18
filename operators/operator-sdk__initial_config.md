# Initial Config for Operator-sdk

## Install Golang binary
~~~
# wget https://dl.google.com/go/go1.12.9.linux-amd64.tar.gz
# tar -zxvf go1.12.9.linux-amd64.tar.gz
# mkdir -p $HOME/projects/src
# vim ~/.bash_profile
export GOPATH=$HOME/projects
export GOROOT=$HOME/go
export PATH=$GOROOT/bin:$PATH
export GO111MODULE=on
# source ~/.bash_profile
~~~

## Install Operator-sdk
~~~
# mkdir $HOME/bin
# RELEASE_VERSION=v0.10.0
# curl -OJL https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}/operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu
# mv operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu $HOME/bin/operator-sdk
# chmod u+x $HOME/bin/operator-sdk
~~~
