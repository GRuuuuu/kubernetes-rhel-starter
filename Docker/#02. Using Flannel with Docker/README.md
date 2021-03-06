# Using Flannel with Docker on RHEL
`OS: RedHat Enterprise Linux 7.4 LE`  
`Architect: IBM Power 8`

## 1. Overview
호스트가 서로 다른 Docker속 컨테이너끼리 네트워크 통신을 가능하게 하는 Flannel 플러그인을 다뤄보겠습니다. 

[참고링크1: Configuring flannel overlay network with VXLAN for Docker on IBM Power Systems servers](https://www.ibm.com/developerworks/library/l-flannel-overlay-network-VXLAN-docker-trs/index.html)  

[참고링크2: ppc64le에서의 flannel 구성 : host 서버 외부로의 docker container network 구성](http://hwengineer.blogspot.com/2018/01/ppc64le-flannel-docker-container.html)

## 2. Prerequisites
- `Docker`가 설치되어 있어야 합니다.  
- `Flannel`빌드에 `golang`이 필요하니, `epel`레포지토리의 추가가 필요합니다.  
- `iptable`의 설치가 필요합니다.
- 노드끼리의 [ssh key교환](https://github.com/GRuuuuu/rhel-starter/tree/master/Openshift/%2301.%20Install%20Openshift%20on%20RHEL#ssh-key-%EA%B5%90%ED%99%98)이 필요
>이 문서에서는 서로다른 <b>두 개</b>의 호스트에 위치한 컨테이너끼리의 통신을 다루고 있습니다.

## 3. Flannel?
### Docker Network 구조 -bridge mode
![Alt text](./img/1.png)  
docker의 기본적인 네트워크 구조는 위와 같습니다. 도커를 설치하고 network interface를 살펴보면 `docker0`라는 virtual interface를 확인할 수 있습니다.   
`docker0`는 일반적인 interface가 아니라 virtual ethernet bridge입니다. bridge는 기본적으로 L2통신 기반이며 container가 하나 생성될때마다 container의 interface가 하나씩 binding됩니다. (그림의 vethXYZ)  
따라서 container가 외부로 통신할때는 반드시 `docker0`를 지나야합니다.  

### 서로 다른 호스트에서의 컨테이너 통신
![Alt text](./img/2.PNG)  
Host1과 Host2는 물리적으로 다른 호스트입니다. 내부에 도커서비스가 돌아가고있고 각각 containerA(123.0.1.1)과 containerB(123.0.1.2)를 서비스하고 있습니다.  
서로 다른 호스트끼리 하나의 컨테이너 클러스터를 구성하고 싶어 containerA와 containerB를 네트워크상으로 연결하는것이 가능할까요?  
결론적으로는 <b>불가능합니다</b>. containerA에 할당된 ip는 123.0.1.1로 docker가 할당해준 ip입니다. 이는 도커호스트 내부에서밖에 사용할 수 없습니다.  
즉 containerA에서 containerB로 ping을 보내려면, (Host1)docker0->eth0->(Host2)eth0->docker0->(ContainerB)eth0의 과정을 통해 전달되어야합니다. 이는 복잡한 포트포워딩이 필요합니다.  
하지만, 런타임으로 인바운드-아웃바운드 포트가 랜덤하게 정해지는 애플리케이션에서는 포트포워딩으로는 해결할 수 없습니다.

### Flannel
![Alt text](./img/3.png)  
Flannel은 한마디로 원래의 패킷을 한번더 감싸는 역할을 하게 됩니다. 그림으로 보면 Node1의 Container1에서 Node2의 Container2로 보내는 패킷을 UDP로 감싸게 됩니다.  
UDP의 헤더에는 출발지와 목적지의 정보가 들어있습니다.  
이를 통해 컨테이너 간 통신을 포트포워딩 없이도 가능하게 합니다.  



## 4. Configure Flannel
### etcd설정
`Flannel`을 구성하기 전에 두대의 서버에 모두 `etcd`의 설치를 진행합니다.

> <b>etcd</b>  
>분산 key/value store로 어떤 노드의 어떤 ID의 컨테이너가 어떤 ip로 할당되어있는지 등등에 관한 정보가 저장되어있습니다. 

~~~sh
$ yum install -y etcd
$ systemctl start etcd
~~~

다음으로는 `Flanneld`의 구성정보를 작성합니다. (두 서버 모두)
~~~json
$ vim flannel-config-vxlan.json

    {
        "Network": "10.172.29.0/16",
        "SubnetLen": 24,
        "Backend": {
            "Type": "vxlan",
            "VNI": 1
        }
    }
~~~
이 파일을 etcd에 집어 넣습니다.
~~~bash
$ etcdctl set /atomic.io/network/config < flannel-config-vxlan.json
~~~
확인은 다음과 같이 진행합니다.
~~~bash
$ etcdctl get /atomic.io/network/config
~~~
이전에 삽입했던 json파일과 동일한 내용이 나온다면 제대로 들어간게 맞습니다.  

etcd가 정상적으로 작동하고 있는지 확인은 다음과 같이 진행합니다.
~~~bash
$ hotname
$ curl -L http://{hostname}:2379/v2/keys/atomic.io/network/config

#결과
{"action":"get","node": {"key":"/atomic.io/network/config","value":"{\n   \"Network\": \"10.172.29.0/16\",\n    \"SubnetLen\": 24,\n    \"Backend\": {\n       \"Type\": \"vxlan\",\n       \"VNI\": 1\n    }\n}\n","modifiedIndex":4,"createdIndex":4}}

$ etcdctl cluster-health

#결과 
member 8e9e05c52164694d is healthy: got healthy result from http://{hostname의 ip}:2379
cluster is healthy
~~~

### Flannel 빌드
직접 빌드해서 사용해 보겠습니다.
>- 버전 1.7 이상의 `golang`을 사용해야 합니다.  
>- `golang`은 `epel`안에 있으므로 반드시 추가해주셔야 합니다.  

flannel을 빌드하기 위해서는 `golang`의 설치가 필요합니다.
~~~bash
$ yum install -y golang
~~~

환경변수는 알아서 세팅됩니다.  
다음으로는, flannel의 repository를 클론받아야 합니다.
~~~bash
#clone받을 폴더를 만들고
$ mkdir -p src/github.com/coreos

#만든 폴더 안에서 클론
$ git clone https://github.com/coreos/flannel.git
~~~
빌드를 시작하기 전에, Makefile을 수정해야 하는데, 그 이유는 default로 amd64를 채택하고 있기 때문입니다. Power환경에서는 ppc64le로 빌드를 해야하기 때문에 수정해줍시다!  

~~~Makefile
#flannel폴더 내부에서
$ vim Makefile


#make tar.gz를 할 것이기 때문에, 다른 영역은 건들이지 말고 하단의 속성들만 바꿔줍니다

#TAG?=$(shell git describe --tags --dirty)
TAG?=v0.11.0-11-ge4deb05-ppc64le
#ARCH?=amd64
ARCH?=ppc64le
#GO_VERSION=1.8.3
GO_VERSION=1.11.5
...
tar.gz:
#ARCH=amd64 make dist/flanneld-amd64
#tar --transform='flags=r;s|-amd64||' -zcvf dist/flannel-$(TAG)-linux-amd64.tar.gz -C dist flanneld-amd64 mk-docker-opts.sh ../README.md
#tar -tvf dist/flannel-$(TAG)-linux-amd64.tar.gz
#ARCH=amd64 make dist/flanneld.exe
#tar --transform='flags=r;s|-amd64||' -zcvf dist/flannel-$(TAG)-windows-amd64.tar.gz -C dist flanneld.exe mk-docker-opts.sh ../README.md
#tar -tvf dist/flannel-$(TAG)-windows-amd64.tar.gz
ARCH=ppc64le make dist/flanneld-ppc64le
tar --transform='flags=r;s|-ppc64le||' -zcvf dist/flannel-$(TAG)-linux-ppc64le.tar.gz -C dist flanneld-ppc64le mk-docker-opts.sh ../README.md
tar -tvf dist/flannel-$(TAG)-linux-ppc64le.tar.gz
#ARCH=arm make dist/flanneld-arm
#tar --transform='flags=r;s|-arm||' -zcvf dist/flannel-$(TAG)-linux-arm.tar.gz -C dist flanneld-arm mk-docker-opts.sh ../README.md
#tar -tvf dist/flannel-$(TAG)-linux-arm.tar.gz
#ARCH=arm64 make dist/flanneld-arm64
#tar --transform='flags=r;s|-arm64||' -zcvf dist/flannel-$(TAG)-linux-arm64.tar.gz -C dist flanneld-arm64 mk-docker-opts.sh ../README.md
#tar -tvf dist/flannel-$(TAG)-linux-arm64.tar.gz
#ARCH=s390x make dist/flanneld-s390x
#tar --transform='flags=r;s|-s390x||' -zcvf dist/flannel-$(TAG)-linux-s390x.tar.gz -C dist flanneld-s390x mk-docker-opts.sh ../README.md
#tar -tvf dist/flannel-$(TAG)-linux-s390x.tar.gz
...
~~~

Makefile을 다 수정하였다면, tar.gz를 만들어 봅시다.
~~~bash
#Makefile이 있는 폴더 내부에서
$ make tar.gz

#특별히 설정을 지정하지 않았다면, dist폴더 내부에 생성됩니다.
$ ls dist |grep tar
    flannel-v0.11.0-11-ge4deb05-ppc64le-linux-ppc64le.tar.gz
~~~

tar파일을 적절한 위치에 풀어놓습니다. 이 문서에서는 예시로 /usr/local/bin에 풀겠습니다.
~~~bash
$ cd /usr/local/bin
$ tar -xvf ~/src/github.com/coreos/flannel/dist/flannel-v0.11.0-11-ge4deb05-ppc64le-linux-ppc64le.tar.gz
    flanneld
    mk-docker-opts.sh
    README.md
~~~

### docker에 flannel붙이기
flanneld구성에 들어가기 전에 docker 서비스를 멈추고 docker0를 삭제합니다.
>`docker0`은 도커가 처음 실행될 때 생성되는 가상 네트워크 인터페이스 입니다. `docker0`에서 지정된 서브넷 안에서 도커 컨테이너의 ip가 private로 할당되고 이를 통해 외부와 연결을 할 수 있습니다.  
>하지만 `docker0`로는 서로다른 물리적인 호스트 안의 컨테이너끼리의 통신을 할수는 없습니다.
~~~bash
$ systemctl stop docker
$ ip link delete docker0
~~~

이제 systemctl에 flanneld서비스를 등록해야 합니다.  
다음과 같은 설정 파일을 만들어 줍니다.
~~~bash
$ vim /lib/systemd/system/flanneld.service
~~~
~~~service
#flanneld.service

[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
EnvironmentFile=-/etc/default/flanneld
ExecStart=/usr/local/bin/flanneld -etcd-endpoints=${FLANNEL_ETCD} -etcd-prefix=${FLANNEL_ETCD_KEY} $FLANNEL_OPTIONS
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
~~~

~~~bash
$ vim /etc/default/flanneld
~~~
~~~conf
#flanneld

# Flanneld configuration options
# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD="http://{hostname의 ip}:2379"
# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_KEY="/atomic.io/network"
# Any additional options that you want to pass
FLANNEL_OPTIONS="-ip-masq=true"
~~~

두개의 파일을 생성하였다면, flanneld의 서비스를 시작합니다.
~~~bash
$ systemctl start flanneld
~~~

정상적으로 작동하고 있는지 확인합니다.
~~~bash
#status가 active상태인지 확인
$ systemctl status flanneld

#생성된 flannel의 정보 확인
$ cat /run/flannel/subnet.env
    FLANNEL_NETWORK=10.172.0.0/16
    FLANNEL_SUBNET=10.172.62.1/24
    FLANNEL_MTU=1450
    FLANNEL_IPMASQ=true

$ ifconfig flannel.1
    flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.172.62.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::9878:9dff:fe3a:ea86  prefixlen 64  scopeid 0x20<link>
        ether 9a:78:9d:3a:ea:86  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 8 overruns 0  carrier 0  collisions 0
~~~
그다음 docker daemon의 설정값을 변경해 줍니다.  
~~~service
#docker service를 중단시키고나서 변경
$ systemctl stop docker

$ vim /etc/systemd/system/multi-user.target.wants/docker.service
#추가
EnvironmentFile=-/run/flannel/subnet.env

#수정
ExecStart=/usr/bin/dockerd \
        --bip=${FLANNEL_SUBNET} \
        --mtu=${FLANNEL_MTU} \
        --iptables=false \
        --ip-masq=false
        --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current \
        --default-runtime=docker-runc \
        --exec-opt native.cgroupdriver=systemd \
        --userland-proxy-path=/usr/libexec/docker/docker-proxy-current
~~~
수정이 완료되었다면, 도커를 재시작합니다.
~~~bash
$ systemctl daemon-reload
$ systemctl start docker
~~~

네트워크가 제대로 할당되었는지 확인해봅시다. 
~~~bash
$ netstat -rn
    여기 flannel과 docker0이 다 떠야되는데 docker0만뜨고 flannel은 안뜸.... 안붙음...
~~~

### container끼리 통신해보기
각 서버에서 컨테이너를 띄우고 서로 통신해봅시다!

~~~bash
$ docker run --rm -it ubuntu:16.04 /bin/bash
~~~
우분투 컨테이너를 각 호스트에서 띄운뒤, 컨테이너 내부에서 핑을 보내봅시다.
~~~bash
#ping을 보내기위한 사전절차
bash$ apt-get update
bash$ apt-get install net-tools
bash$ apt-get install iputils-ping

#핑
bash$ ping {다른서버의 컨테이너속 eth0 addr}
~~~

<성공한사진이 들어올 예정>

----