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
3. 웹서버 설치/구성 (httpd)
4. DNS서버 설치/구성 (bind)
5. 로드 밸런서 설치/구성 (haproxy)
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
- OpenShift Pull Secrect을 복사해서 별도의 파일에 저장해 둡니다. <p>
  https://cloud.redhat.com/openshift/install/power/user-provisioned <p>
  ![RedHat_Login](https://anniecode.github.io/blog/assets/images/redhat_login1.png) <p>
  ![Power](https://anniecode.github.io/blog/assets/images/power1.png) <p>
  ![Copy Pull Secret](https://anniecode.github.io/blog/assets/images/PullSecret.png) <p>
- Red Hat 사이트에서 설치에 필요한 파일을 인프라 노드에서 다운로드 <p>
  - RHCOS 이미지 파일 <br>
  ```
  $ wget https://mirror.openshift.com/pub/openshift-v4/ppc64le/dependencies/rhcos/4.4/4.4.9/rhcos-4.4.9-ppc64le-metal.ppc64le.raw.gz
  ```
  - OpenShift Installer 바이너리 <br>
  ```
  $ wget https://mirror.openshift.com/pub/openshift-v4/ppc64le/clients/ocp/4.3.18/openshift-install-linux-4.3.18.tar.gz -C /usr/local/bin
  ```
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

**로드 밸런서**
OpenShift 배포 전에 2대의 L4 밸런서가 필요합니다. 1대는 API용이고, 다른 1대는 Ingress Controller용인데, 테스트 목적으로는 1대로도 가능합니다.

```
$ yum install haproxy
$ vi /etc/haproxy/haproxy.cfg
```
