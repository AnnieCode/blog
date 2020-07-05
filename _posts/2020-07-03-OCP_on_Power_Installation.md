---
title: "OpenShift Container Platform 설치하기"
date: 2020-07-03 11:00:00 +0900
categories: Kubernetes
tags: OpenShift
sidebar:
  nav: "main"
---

### IBM Power Systems 에 OpenShift Container Platform 4.4.9 설치하기

얼마전에 나온 OCP 4.4.9 를 IBM Power에 설치해 보았습니다.
전반적으로 OpenShift Documentation 사이트의 정보가 도움이 되었지만, PowerVM 환경에 대한 고려사항이 나와 있지 않아 다소 애를 먹었습니다.

#### 인터넷 및 텔레메트리 액세스 확인
OCP를 설치하려면, 설치할 노드에서 외부 인터넷 액세스가 가능한지 확인해야 합니다. 일단 설치 프로그램을 인터넷에서 다운로드 받아야 하고, 서브스크립션 관리 작업과 클러스터 설치/업데이트에 필요한 패키지를 다운로드 받아야 하므로 인터넷 액세스가 가능해야 합니다.  
또한 텔레메트리라는 클러스터 상태와 업데이트 성공 여부를 측정하는 서비스를 위해서도 인터넷 액세스가 필요합니다.



\* Bootstrap 은 OCP 클러스터를 설치할 때만 필요하고, 설치가 완료된 이후에는 필요 없으므로 지워도 됩니다.

Bootstrap, Master, Worker 노드에는 Red Hat Enterprise Linux CoreOS(RHCOS)가 설치됩니다.

#### 네트워크 요구사항

모든 RHCOS 노드들은 부팅하면서 initramfs 에 명시된 네트워크에 액세스해서 ignition config 파일을 가져옵니다. 따라서 초기 부팅할 때 DHCP 서버가 준비되어 있거나, DHCP 서버가 없을 경우 Static IP address가 설정되어 있어야 합니다.





#### Certificate signing request (CSR) 관리

클러스터가 자동으로 Worker 노드 추가하는 데 일부 제약이 있기 때문에, 설치 마지막 단계에서 클러스터 Certificate Signing Request (CSR) 을 승인하는 작업이 필요합니다. 사용자 배포 인프라 환경(User Provisioned Infrastructure)에서는 `machine-approver`가 신청중인 머신의 인증서가 유효한지 확인할 수 없기 때문이라고 합니다.

#### 사용자 배포 인프라 환경 구축 (User Provisioned Infrastructure:UPI)

OpenShift Container Platform 클러스터를 설치/구축하기 전에 먼저 준비해야 할 일이 있습니다. 사실 작업해보니 클러스터 설치/구축하는 과정보다 인프라 환경 구축하는 일이 더 많은 공이 들어갔습니다. 여러분들도 직접 해보시면 무슨 말인지 이해가 되실 껍니다.

인프라 환경 구축을 위한 인프라 노드 (또는 Helpernode)를 준비하십시오.
저는 RHEL8 기반의 VM을 인프라 노드로 구성하였습니다.

**작업순서**
1. 설계 작업 (VM, 네트워크)
2. VM(LPAR) 생성
3. DNS서버 설치/구성 (bind)
4. 로드 밸런서 설치/구성 (haproxy)
5. 웹서버 설치/구성 (httpd)
6. NFS서버 설치/구성

---
### 1. 설계 작업 (VM, 네트워크)

#### 프라이빗 클라우드 인프라에 OCP 클러스터를 구성하기 위한 인프라 조건

|노드유형|노드수|
|:--------:|:------:|
|Infra 노드|1|
|Bootstrap 노드|1|
|Master 노드|3|
|Worker 노드|2|

#### 노드별 최소 요구사항

각 클러스터 노드의 HW 최소 요구사항

|노드|운영체제|vCPU|RAM|Storage|노드수|
|:-------:|:---------:|:-------:|:--------:|:---------:|:----:|
|Infra (Helper)|RHEL8|1|4 GB|30 GB|1|
|Bootstrap|RHCOS|2|16 GB|120 GB|1|
|Master|RHCOS|2|16 GB|120 GB|3|
|Worker|RHCOS|2|8 GB|120 GB|2|


>**주의사항** : IBM Power 서버는 PowerVM 환경에서 기본 SMT8 모드를 지원하므로, 설정된 가상 코어수 (Virtual Processor)에 8배수를 한 개수가 vCPU로 인식됩니다. 예를 들어, VP 를 1로 설정했다면, RHCOS에서는 8개의 Logical CPU 가 보입니다. 마스터 노드의 경우, vCPU 개수가 많으면 커널 메모리를 많이 필요로 하므로 초기 구성시 마스터 노드의 VP개수를 1이나 2로 설정하는 것을 권장합니다.

##### 실제 테스트한 VM 환경

![Cluster_Architecture](https://anniecode.github.io/blog/assets/images/Cluster_Architecture1.png)


#### 클러스터 설계에 필요한 사전 정보 수집/설정

- 클러스터 이름 : `____________________________` (예:mycluster)
- 도메인 네임 : `____________________________` (예:example.com)
- OpenShift Pull Secrect을 복사해서 별도의 파일에 저장해 둡니다.
  https://cloud.redhat.com/openshift/install/power/user-provisioned
  ![RedHat_Login](https://anniecode.github.io/blog/assets/images/redhat_login1.png)
  ![Power](https://anniecode.github.io/blog/assets/images/power1.png)
  ![Copy Pull Secret](https://anniecode.github.io/blog/assets/images/PullSecret.png)
- Red Hat 사이트에서 설치에 필요한 파일 다운로드
  - 각 VM 부팅에 사용할 RHCOS installer iso 파일 다운로드 (이후에 OCP 클러스터의 각 노드의 Virtual Optical로 구성할 VIOS 서버에 저장/구성할 것입니다)
  ```
  $ wget https://mirror.openshift.com/pub/openshift-v4/ppc64le/dependencies/rhcos/4.4/4.4.9/rhcos-4.4.9-ppc64le-installer.ppc64le.iso
  ```
---


#### 사용자 배포 인프라에 필요한 네트워크 요구사항

모든 Red Hat Enterprise Linux CoreOS (RHCOS) 머신은 처음 설치할 때 부팅 과정에서 Machine Config 서버로부터 Ignition config 파일을 가져와야 하는데, 이를 위한 네트워크 구성을 initramfs 옵션을 통해 설정합니다.

따라서 초기 부팅 과정에서 클러스터를 구성하는 각 호스트에 DHCP 서버나 정적 IP 주소가 설정되어 있어야 합니다.

장기적인 측면에서 클러스터의 머신을 관리하기에는 DHCP 서버를 사용하는 것을 권장합니다만, 저는 여건상 정적 IP 주소를 사용하였습니다.

클러스터 설치/구성이 끝나고 나면, 각 마스터 노드의 Kubernetes API 서버가 클러스터 머신의 노드이름(호스트명)을 인식하고 각 머신 간에 호스트 이름을 통한 통신을 위해 DNS 서버가 필요합니다.

표1. 모든 머신 <-> 모든 머신간 네트워크 요구사항

|프로토콜|포트|설명|
|:-------:|:------|:-------------------------------|
|ICMP|N/A|네트워크 테스트|
|TCP|9000-9999|호스트 서비스|
|TCP|10250-10259|쿠버네티스 디폴트 포트|
|TCP|10256|openshift-sdn|
|UDP|4789|VXLAN과 GENEVE|
|UDP|6081|VXLAN과 GENEVE|
|UDP|9000-9999|호스트 서비스|
|TCP/UDP|30000-32767|쿠버네티스 노드포트|

표2. 모든 머신 <-> 마스터 노드

|프로토콜|포트|설명|
|:-------:|:------|:-------------------------------|
|TCP|2379-2380|etcd 서버, peer, metrics 포트|
|TCP|6443|쿠버네티스 API|

#### 네트워크 토폴로지 요구사항

> **주의사항** Openshift Container platform 의 모든 노드는 플랫폼 컨테이너 이미지를 가져오고 Red Hat에 텔레메트리 데이터를 보내기 위해 인터넷 액세스가 가능해야 합니다.

**DNS 서버**

```
$ yum install bind
```

`/etc/named.rfc1912.zones` 파일에 아래 내용 추가
```
zone "mycluster.example.com" IN {
        type master;
        file "mycluster.example.com.zone";
        allow-update { none; };
};

zone "14.10.10.in-addr.arpa" IN {
        type master;
        file "mycluster.example.com.rr";
        allow-update { none; };
};
```

```
$ systemctl enable bind
$ systemctl start bind
```

**로드 밸런서**
OpenShift 배포 전에 2대의 L4 밸런서가 필요합니다. 1대는 API용이고, 다른 1대는 Ingress Controller용인데, 테스트 목적으로는 1대로도 가능합니다.

```
$ yum install haproxy
$ vi /etc/haproxy/haproxy.cfg
```
```
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   http://haproxy.1wt.eu/download/1.4/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2        info

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                  tcp
    log                     global
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

##
#  balancing for OCP Kubernetes API Server
##
frontend openshift-api-server
    bind *:6443
    mode tcp
    option tcplog
    default_backend openshift-api-server


backend openshift-api-server
    balance source
    mode tcp
    server bootstrap 10.10.14.126:6443 check
    server master0 10.10.14.117:6443 check
    server master1 10.10.14.118:6443 check
    server master2 10.10.14.119:6443 check
##
# balancing for OCP Machine Config Server
##
frontend machine-config-server
    bind *:22623
    mode tcp
    option tcplog
    default_backend machine-config-server

backend machine-config-server
    balance source
    mode tcp
    server bootstrap 10.10.14.126:22623 check
    server master0 10.10.14.117:22623 check
    server master1 10.10.14.118:22623 check
    server master2 10.10.14.119:22623 check

##
# balancing for OCP Ingress Insecure Port & Admin Page
##
frontend ingress-http
    bind *:80
    mode tcp
    option tcplog
    default_backend ingress-http

backend ingress-http
    balance source
    mode tcp
    server worker1 10.10.14.122:80 check
    server worker2 10.10.14.123:80 check
    server worker3 10.10.14.124:80 check

##
# balancing for OCP Ingress Secure Port
##
frontend ingress-https
    bind *:443
    mode tcp
    option tcplog
    default_backend ingress-https

backend ingress-https
    balance leastconn
    mode tcp
    server worker1 10.10.14.122:443 check
    server worker2 10.10.14.123:443 check
    server worker3 10.10.14.124:443 check
```
```
$ systemctl enable haproxy
$ systemctl start haproxy
```

**웹서버**
각 노드가 부팅시 네트웍으로 Ignition File과 RHCOS 이미지를 다운받을 수 있도록 HTTP로 해당 파일을 서비스해줄 웹서버가 필요합니다.
설치 및 시스템 부팅시 자동 시작되도록 구성합니다.
```
$ yum install -y httpd
$ systemctl enable httpd
```
html root디렉토리는 /var/www/html입니다.
이 디렉토리에 파일을 하나 만들고 제대로 접근되는지 테스트합니다.
```
$ cd /var/www/html
$ echo "hello httpd" > README.md
$ curl http://10.10.14.90/README.md
```

**NFS 서버**

```
# 설치
$ yum install -y nfs-utils

# Volume 디렉토리 생성
$ df -h
Filesystem                 Size  Used Avail Use% Mounted on
devtmpfs                   4.0G     0  4.0G   0% /dev
tmpfs                      4.0G  128K  4.0G   1% /dev/shm
tmpfs                      4.0G   46M  4.0G   2% /run
tmpfs                      4.0G     0  4.0G   0% /sys/fs/cgroup
/dev/mapper/rootvg-lvroot   47G  9.2G   38G  20% /
/dev/dm-4                  500G  3.6G  497G   1% /data
/dev/dm-3                 1014M  189M  826M  19% /boot
tmpfs                      812M     0  812M   0% /run/user/0
$ mkdir -p /data/user
$ mkdir -p /data/images
$ chmod -R 777 /data

# /etc/exports파일에 volume 디렉토리 추가
$ vi /cat/exports

/data/images *(rw,root_squash,all_squash,no_wdelay)
/data/user *(rw,root_squash,all_squash,no_wdelay)

# NFS 서버 시작
$ systemctl enable nfs-server
$ systemctl start nfs-server
$ systemctl status nfs-server
```

**Infra#1(Bastion) 노드에서 SSH Key 생성**
```
$ ssh-keygen -b 4096 -t rsa
```
현재 사용자의 HOME디렉토리 하위에 '.ssh'라는 디렉토리가 생깁니다.
그 디렉토리에 private key파일인 id_rsa와 public key파일인 id_rsa.pub가 생성됩니다.


**OpenShift Installer 다운로드 및 설치**
```
https://mirror.openshift.com/pub/openshift-v4/ppc64le/clients/ocp/4.4.9/openshift-install-linux-4.4.9.tar.gz
$ wget https://mirror.openshift.com/pub/openshift-v4/ppc64le/clients/ocp/4.4.9/openshift-install-linux-4.4.9.tar.gz 
$ tar -xvf openshift-install-linux-4.4.9.tar.gz
$ openshift-install version
openshift-install 4.4.9
built from commit 1541bf917973186bbab6a5f895f08db4334a5d9a
release image quay.io/openshift-release-dev/ocp-release@sha256:ae48474c6fcd0666a672ce2a449736a2549693c04186ea588e37477635c976a6
```

**CLI 툴킷 다운로드 및 설치**
```
$ wget https://mirror.openshift.com/pub/openshift-v4/ppc64le/clients/ocp/4.4.9/openshift-client-linux-4.4.9.tar.gz
$ tar -xvf openshift-client-linux-4.4.9.tar.gz
$ cp ./kubectl /usr/local/bin/kubectl
$ cp ./oc /usr/local/bin/oc

$ oc version
Client Version: 4.4.9

$ kubectl version
Client Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.0-4-g38212b5", GitCommit:"d038424d6d4f1cc39ad586ac0d36dac3a7a37ceb", GitTreeState:"clean", BuildDate:"2020-06-16T03:31:01Z", GoVersion:"go1.13.4", Compiler:"gc", Platform:"linux/ppc64le"}
```

**인스톨 작업을 진행할 디렉토리와 install-config.yaml 생성**

```
$ mkdir install
$ cd install
$ vi install-config.yaml
apiVersion: v1
baseDomain: example.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 2
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: haru
networking:
  clusterNetwork:
  - cidr: 10.136.0.0/14
    hostPrefix: 23
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.36.0.0/16
platform:
  none: {}
fips: false
pullSecret: '{"auths": ...}'
sshKey: 'ssh-rsa ... '
```
`baseDomain: ` 클러스터 도메인 이름으로 변경
`pullSecret: ` 다음 부분에 이전에 Red Hat 사이트에서 복사했던 Pull Secret 을 붙여넣기 
`sshKey: ` 다음 부분에 이전에 생성했던 public SSH key 를 복사해서 붙여넣기

OpenShift Installer로 Kubernetes Manifest과 Ignition 파일을 생성하면, install-config.yaml 파일이 삭제되므로, 이후 보관을 위해 백업을 받아 놓습니다. 
```
$ cp install-config.yaml install-config.yaml.bak
```

**Manifest 과 Ignitaion 파일 생성**
$ ./openshift-install create manifests --dir=install/
INFO Consuming Install Config from target directory
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings

$ ls install/
manifests  openshift

# masterSchedulable 을 False로 수정
$ vi manifests/cluster-scheduler-02-config.yml
```

**Ignition 파일 작업**
``
$ ./openshift-install create ignition-configs --dir=install/
INFO Consuming OpenShift Install (Manifests) from target directory
INFO Consuming Worker Machines from target directory
INFO Consuming Master Machines from target directory
INFO Consuming Openshift Manifests from target directory
INFO Consuming Common Manifests from target directory

$ ls install/
auth  bootstrap.ign  install-config.yaml.bak  master.ign  metadata.json  worker.ign

# Ignition 파일을 웹서버 디렉토리로 복사하고 권한 변경
$ cp dir/*.ign /var/www/html
$ chmod 644 /var/www/html/*.ign
```

**RHCOS raw 이미지 파일 작업**
```
$ cd /var/www/html/
$ wget https://mirror.openshift.com/pub/openshift-v4/ppc64le/dependencies/rhcos/4.4/4.4.9/rhcos-4.4.9-ppc64le-metal.ppc64le.raw.gz
```

[참조] Red Hat OpenShift Documentation 
https://docs.openshift.com/container-platform/4.4/installing/installing_ibm_power/installing-ibm-power.html

