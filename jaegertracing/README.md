This guide provides the steps for anyone who is interested in getting started with end to end tracing using Jaeger and Opentracing libraries. 
## Jaeger & Opentracing : 
Jaeger along with Opentracing provides a standard solution for tracing the complex background transactions in Ceph. 
Opentracing is the framework which in this case works with Jaeger backend, consisting of an agent which listens for any span, upon receiving one sends them to the collector. 
The spans gathered by the collectors are weived together to form traces, which can then be rendered on Jaeger UI. 
![Architecture](https://www.jaegertracing.io/img/architecture-v1.png)
## Steps: 
You may want to use a preconfigured container image, instead of the local build.
The instructions work ideally for Ubuntu 16.04, have been tested on Fedora 27 as well. 

## Installing Jaeger dependencies :
https://ubuntu.pkgs.org/17.04/ubuntu-universe-amd64/nlohmann-json-dev_2.1.1-1.1_all.deb.html

### Thrift 0.11.0 
```

sudo apt-get install automake bison flex g++ git libboost-all-dev libevent-dev libssl-dev libtool make pkg-config
git clone https://github.com/apache/thrift.git && cd thrift && git checkout 0.11.0 
./bootstrap.sh 
./configure --with-boost=/usr/local 
make 
sudo make install

```
### Opentracing-cpp
```
git clone https://github.com/opentracing/opentracing-cpp.git \
&& cd opentracing-cpp/ 
mkdir .build 
cd .build 
cmake .. 
make -j$(nproc)
sudo make install
```

### Jaeger-client-cpp 
```
git clone https://github.com/jaegertracing/jaeger-client-cpp.git && cd jaeger-client-cpp
mkdir build
cd build 
cmake -DBUILD_TESTING=OFF ..
make
sudo make install
   
```
### Install docker(if not present)
```
sudo apt-get update \
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common \
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add - \
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
sudo apt-get update \
sudo apt-get install docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker
sudo usermod -aG docker $(whoami)

```
### Pull the jaeger UI docker image
```
docker run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HTTP_PORT=9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.12

```
### Pull the work in progress branch
```
git clone https://github.com/ceph/ceph.git 
cd ceph 
git fetch
git checkout wip-jaegertracing-in-ceph
./install-deps.sh && ./do_cmake.sh
cd build
make vstart -j$(nproc)

```

I test if the setup is working fine or not by performing a write operation in RADOS bench, use the following commands :
```
// creating a pool
    $ bin/ceph osd pool create test 8
// writing to it
    $ bin/rados -p test bench 5 write --no-cleanup
```
You can then navigate to http://localhost:16686 to access the Jaeger UI.

### [WIP] RGW tracing

Can run a simple span test using the amazon S3 API.

#### Pull the WIP branch
```
git clone https://github.com/ceph/ceph.git 
cd ceph 
git fetch
git checkout wip-jaegertracing-in-rgw
./install-deps.sh && ./do_cmake.sh
cd build
make vstart -j$(nproc)

```
#### Install AWS CLI

1) Using pip to install aws CLI

```
sudo pip install --upgrade pip
sudo pip install awscli
sudo pip install awscli --upgrade
```
2) Using the bundeled installer

```
curl "https://s3.amazonaws.com/aws-cli/awscli-bundle.zip" -o "awscli-bundle.zip"
unzip awscli-bundle.zip
sudo ./awscli-bundle/install -i /usr/local/aws -b /usr/local/bin/aws
```

check the version using 

```
/usr/local/bin/aws --version
```

#### Configuring aws

Run the commands in one of your Ceph cluster nodes with access to cluster for administration.

```
sudo radosgw-admin user create --uid="tester" --display-name="tester S3User
```
From the output, note the ```access-key``` and ```secret-key```. 

Before you can start using AWS CLI tool to interact with AWS services, you need to configure it by running the aws configure command.

```
aws configure --profile=ceph
```
When promperd, enter the secret key and access key obtained in the prior step.

You need to have the IP address of one of your Rados Gateway nodes or equivalent DNS name configured. Here, we assume a dummy rgw-server's URL to be http://172.21.148.53

```
aws --profile=ceph --endpoint=http://172.21.148.53 s3 mb s3://test_bucket
```

The above command will create a bucket called ```test_bucket```.

To get a sample trace, just list all buckets using 

```
sudo radosgw-admin bucket list
```

You can then navigate to http://localhost:16686 to access the Jaeger UI.


## Optional Build from Source

### Yaml-Cpp

```
git clone https://github.com/jbeder/yaml-cpp.git && cd yaml-cpp 
mkdir build
cd build
cmake -DBUILD_SHARED_LIBS=ON ..
make 
sudo make install
```

### nlohmann-json

```
$ git clone https://github.com/nlohmann/json.git \
    && cd json \
    && mkdir build \
    && cd build \
    && cmake -DBUILD_SHARED_LIBS=ON .. \
    && make -j$(nproc) \
    && sudo make install
```
