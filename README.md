# docker_open5gs
Docker files to build and run open5gs in a docker

## Tested Setup

Docker host machine

- Ubuntu 18.04 and 20.04

SDRs tested with srsLTE eNB

- Ettus USRP B210
- LimeSDR Mini v1.3

UERANSIM (gNB + UE) simulator

## Build and Execution Instructions

* Mandatory requirements:
	* [docker-ce](https://docs.docker.com/install/linux/docker-ce/ubuntu)
	* [docker-compose](https://docs.docker.com/compose)

Install Docker: 

```
sudo apt install -y git net-tools putty
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu  $(lsb_release -cs)  stable"
sudo apt update
sudo apt install -y docker docker-ce

# Add your username to the docker group, otherwise you will have to run in sudo mode.
sudo usermod -a -G docker $(whoami)
reboot

# https://docs.docker.com/compose/install/
sudo curl -L "https://github.com/docker/compose/releases/download/v2.12.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

```

Clone repository and build base docker image of open5gs, kamailio, ueransim

```
git clone https://github.com/herlesupreeth/docker_open5gs
cd docker_open5gs/base
docker build --no-cache --force-rm -t docker_open5gs .

cd ../ims_base
docker build --no-cache --force-rm -t docker_kamailio .

cd ../srslte
docker build --no-cache --force-rm -t docker_srslte .

cd ../ueransim
docker build --no-cache --force-rm -t docker_ueransim .

cd ../oai
docker build --no-cache --force-rm -t docker_oaienb .
```

### Build and Run using docker-compose

```
cd ..
set -a
source .env
docker-compose build --no-cache
docker-compose up

# srsRAN eNB
docker-compose -f srsenb.yaml up -d && docker attach srsenb
# srsRAN gNB
docker-compose -f srsgnb.yaml up -d && docker attach srsgnb
# srsRAN ZMQ based setup
    # eNB
    docker-compose -f srsenb_zmq.yaml up -d && docker attach srsenb_zmq
    # gNB
    docker-compose -f srsgnb_zmq.yaml up -d && docker attach srsgnb_zmq
    # 4G UE
    docker-compose -f srsue_zmq.yaml up -d && docker attach srsue_zmq
    # 5G UE
    docker-compose -f srsue_5g_zmq.yaml up -d && docker attach srsue_5g_zmq

# UERANSIM gNB
docker-compose -f nr-gnb.yaml up -d && docker attach nr_gnb

# UERANSIM NR-UE
docker-compose -f nr-ue.yaml up -d && docker attach nr_ue

# OAI ENB
docker-compose -f oaienb.yaml up -d && docker attach oaienb && \
. ./oaienv && /mnt/oai/oai_init.sh && cd cmake_targets/ran_build/build && \
./lte-softmodem -O $OPENAIR_DIR/targets/PROJECTS/GENERIC-LTE-EPC/CONF/enb.band7.tm1.50PRB.usrpb210.conf -d

```

## Configuration

For the quick run (eNB/gNB, CN in same docker network), edit only the following parameters in .env as per your setup

```
MCC
MNC
TEST_NETWORK --> Change this only if it clashes with the internal network at your home/office
DOCKER_HOST_IP --> This is the IP address of the host running your docker setup
SGWU_ADVERTISE_IP --> Change this to value of DOCKER_HOST_IP set above only if eNB/gNB is not running the same docker network/host
UPF_ADVERTISE_IP --> Change this to value of DOCKER_HOST_IP set above only if eNB/gNB is not running the same docker network/host
```

If eNB/gNB is NOT running in the same docker network/host as the host running the dockerized Core/IMS then follow the below additional steps

Under mme section in docker compose file (docker-compose.yaml, nsa-deploy.yaml), uncomment the following part
```
...
    # ports:
    #   - "36412:36412/sctp"
...
```

Under amf section in docker compose file (docker-compose.yaml, nsa-deploy.yaml, sa-deploy.yaml), uncomment the following part
```
...
    # ports:
    #   - "38412:38412/sctp"
...
```

If deploying in SA mode only (sa-deploy.yaml), then uncomment the following part under upf section
```
...
    # ports:
    #   - "2152:2152/udp"
...
```

If deploying in NSA mode only (nsa-deploy.yaml, docker-compose.yaml), then uncomment the following part under sgwu section
```
...
    # ports:
    #   - "2152:2152/udp"
...
```

## Register a UE information

Open (http://<DOCKER_HOST_IP>:3000) in a web browser, where <DOCKER_HOST_IP> is the IP of the machine/VM running the open5gs containers. Login with following credentials
```
Username : admin
Password : 1423
```

Using Web UI, add a subscriber

## srsLTE eNB settings

If SGWU_ADVERTISE_IP is properly set to the host running the SGWU container in NSA deployment, then the following static route is not required.
On the eNB, make sure to have the static route to SGWU container (since internal IP of the SGWU container is advertised in S1AP messages and UE wont find the core in Uplink)

```
# NSA - 4G5G Hybrid deployment
ip r add <SGWU_CONTAINER_IP> via <SGWU_ADVERTISE_IP>
```

## Not supported
- IPv6 usage in Docker

