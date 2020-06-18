+++
title = "SR-IOV and DPDK packet processing acceleration in kubernetes"
description = ""
tags = [
    "SR-IOV",
    "DPDK",
    "Kubernetes",
]
date = "2019-07-01"
categories = [
    "Data plane acceleration",
    "Kubernetes",
]
highlight = "true"
+++


Fast data path를 위한 DPDK와 SR-IOV를 Container와 Kubernetes에 적용하는 방법을 알아보자

## Container에 적용하기

오픈소스 CNI plug-ins을 이용하여 container내에서 fast data path networking을 가능하게 할 수 있다.
여기서는 두개의 SR-IOV interfaces와 SSH 원격 접속을 위한 하나의 Management interface를 적용한다.

DPDK 기반의 애플리케이션을 실행하기 위해
1. SR-IOV VF가 enabled되어 있어야 하고 
2. CNI plug-ins으로 Docker container에 노출 시켜야 하고
3. Hugepages가 host machine에 enabled되어야 한다. 

일반적인 Linux distro가 지원하는 Hugepage sizes는 2MB와 1GB 이다. 만약 1GB Hugepages를 설정해야 한다면 main memory 단편화 때문에 Host의 kernel boot parameters를 설정하여 boot time에 Hugepages가 설정하여야 한다.
Kernel boot parameters는 /etc/default/grub를 수정하고 update-grub 를 실행하면 된다. 
예를 들어 GRUB_CMDLINE_LINUX="default_Hugepagesz =1G Hugepagesz=1G Hugepages=4" 이렇게 수정하고 GRUB를 update하고 reboot한 다음에 아래처럼, Hugepages가 이용가능한지 확인 한다.

```
$ cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-4.4.0-134-generic default_Hugepagesz=1G Hugepagesz=1G Hugepages=4
$ cat /proc/meminfo | grep -i Hugepages_
Hugepages_Total: 4
Hugepages_Free: 4
Hugepages_Rsvd: 0
Hugepages_Surp: 0
Hugepagesize: 1048576 kB
```

Hugepages는 host에 필수로 mount되어 있어야 한다. Ubuntu 16.04는 /dev/Hugepages에 자동으로 마운트 한다. 
그리고 boot parameters를 이용하여 DPDK application을 위해 사용되는 CPU cores를 아래처럼 설정한다.
GRUB_CMDLINE_LINUX="default_Hugepagesz=1G Hugepagesz=1G Hugepages=4 isolcpus=4,5,6,7,8,9"
DPDK는 UIO, VFIO등 Linux kernel drivers를 지원하는데 여기서는 VFIO이 보다 안정적이므로 이것을 사용한다.
VFIO kernel module은 OS에서 제공되며 Intel VT-d 가상화가 BIOS에서 enabled되어야 하고 IOMMU는 kernel boot parameter로 꼭 설정되어야 한다.

```
$ cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz -4.4.0-134-generic default_Hugepagesz=1G Hugepagesz=1G Hugepages=4 intel_iommu=on
```

enable될 VFs의 개수는 ‘sys’ virtual file system인 /sys/class/net/$IFNAME/device/sriov_numvfs 를 이용하여 설정할 수 있다
MAC address는 아래와 같은 명령어로 각 VF에 설정 한다.
link set dev $IFNAME vf $i mac ff:ff:ff:ff:ff:ff ($1 은 VF번호)

Docker container가 생성될 때, network 설정 파일이 있는 /etc/cni/net.d/와 /opt/cni/bin에 있는 plug-in의 binary를 찾는다. 
여기서는 DPDK를 지원하는 SR-IOV CNI plug-in을 사용하였다.

base image Ubuntu 16.04에 관련된 libraries와 tools을 설치하여 DPDK 애플리케이션 container를 build하였다.

```
$ git clone http://dpdk.org/git/dpdk dpdk
$ cat > build-dpdk.sh<<EOF
echo "Build DPDK"
make install T="\$RTE_TARGET" EXTRA_CFLAGS="-g-O0" -j8
if [ \$? != 0 ]; then
     echo -e "\tBuilding DPDK failed"
     exit 1
fi
cd app/test -pmd/
make -j8
EOF
$ chmod +x build -dpdk.sh
$ cat > Dockerfile <<EOF
FROM ubuntu :16.04
RUN apt-get update; apt-get install -y apt-utils build-
      essential less make kmod vim pciutils libnuma-dev python
      gdb linux -headers -$(uname -r);
COPY ./dpdk /root/dpdk
COPY ./build -dpdk.sh /root/dpdk/build -dpdk.sh
WORKDIR /root/dpdk/
ENV RTE_SDK "/root/dpdk"
ENV RTE_TARGET "x86_64 -native -linuxapp -gcc"
RUN ./build -dpdk.sh
EOF
$ docker build -t dpdk-docker:latest .
```

각 SR-IOV VF는 container에 적용된다. CNI plug-in 설정은 아래처럼 한다.

```
$ cat /etc/cni/net.d/10-sriov-dpdk.conf
{
      "name": "net1",
      "type": "sriov",
      "if0": "enp1s0f1",
      "if0name": "if0",
      "dpdk": {
             "kernel_driver":"ixgbevf",
             "dpdk_driver":"igb_uio",
             "dpdk_tool":"\$RTE_SDK/usertools/dpdk-devbind.py"
      }
}
```

“type”은 sriov 이다. 
“if0”은 SR-IOV interface또는 PF의 이름이다. 
“if0name”은 container 내부 및 DPDK parameters 내의 이름이다.
"kernel_driver"는 인터페이스가 사용중인 현재 커널 드라이버이다. 
"dpdk_driver"는 사용 되어질 DPDK driver이다.
"dpdk_tool"은 인터페이스를 DPDK에 바인딩하는데 사용할 수 있는 DPDK 소스 코드와 함께 제공되는 도구이다. 
두 번째 SR-IOV VF를 위한 설정은 위의 구성과 유사하게 하면 된다.

컨테이너에서 DPDK 응용 프로그램을 실행하려면 인터페이스를 바인드하고 Hugepage 정보를 얻을 수 있도록 Docker의 --privileged 플래그를 적용해야 한다. 
또한, Hugepage 마운트 위치는 볼륨으로 컨테이너에 마운트 되어야 한다. 도커 실행 명령은 다음과 같다.

```
docker run -it -v /dev/Hugepages:/dev/Hugepages --privileged dpdk-docker:gdb /bin/bash
```

## OVS-DPDK를 이용하여 Container에 SR-IOV와 DPDK 적용하기
좀 더 현실적인 시나리오에서는 Host에는 여러 containers가 있을 수 있고 Host간에 전달되는 패킷이 있을 수 있다.
Docker에서 제공하는 "Docker0" bridge는 한 host내에서 작동하고 Docker의 오버레이 네트워크 드라이버와 연동하는 Kubernetes third part 네트워크 plug-ins은(Flannel, Weave Net, Calico 등) DPDK나 SR-IOV VF 같은 fast data path를 지원하지 않는다. 
Open vSwitch (OvS)는 DPDK를 지원하며 Open Flow를 사용하여 네트워크를 쉽게 구성 할 수 있는 방법을 제공한다. OvS는 오버레이 네트워크를 구성하기 위한 터널링 인터페이스를 사용하기 때문에 복잡한 토폴로지를 쉽게 확장할 수 있다.
### OvS build
OvS Release 페이지에서 사용하는 DPDK 버전(2.9.x), Linux 커널 버전(4.4.0-134-generic)에 호환되는 OvS 버전(17.11.3)을 다운 받는다.
환경 변수 RTE_SDK(소스 코드의 경로) 및 RTE_TARGET(x86_64-native-linuxapp-gcc)을 설정해야한다.
그런 다음 OvS는 --with-dpdk = "$RTE_SDK / $RTE_TARGET" flag를 넣고 소스를 컴파일 한다.
### OvS 설정
lcore, memory 및 기타 DPDK parameter관련 설정을 OvS 데이터베이스에 설정한다.

```
sudo ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk -init=true
sudo ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk -lcore-mask=0x0F
sudo ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk -socket-mem=256
sudo ovs-vsctl --no-wait set Open_vSwitch . external-ids: system -id="${HOSTNAME}id" 
```

위의 시스템 식별자 인 system-id는 OvS와 오버레이 네트워크를 설정할 때 사용한다.
OvS bridge와 DPDK를 사용할 것이다. 인터페이스는 DPDK에 바인딩 된 다음 OvS 브리지에 연결되고 한 쌍의 flows가 브리지에 추가되어 단순히 한 interface에서 수신 된 패킷은 다른 interface로 전달 된다. 
다음과 같다.
```
$ ovs-vsctl add-br dpdk-br -- set bridge dpdk-br datapath_type=netdev
$ ovs-vsctl add-port dpdk-br p1 -- set Interface p1 type=dpdk options:dpdk-devargs=0000:01:00.0 ofport_request=1
$ ovs-vsctl add-port dpdk-br p2 -- set Interface p2 type=dpdk options:dpdk-devargs=0000:01:00.1 ofport_request=2
$ ovs-ofctl add-flow ext1 in_port=2,action=output:1 $ ovs-ofctl add-flow ext1 in_port=1,action=output:2
```


## Kubernetes에 Fast data path 적용
현재는 Pod가 생성될때 요구하는 DPDK Interfaces개수를 참조하여 Worker node를 선택할 수 있는 Scheduling 기법이 없다. 그래서 별도의 custom scheduler를 구현하는것이 바람직하다. custom scheduler 구현 방법은 다음에 정리할 예정이다.
일단, 모든 worker 노드가 DPDK 지원 NIC를 가지고 있다고 가정하자.

Pod는 DPDK interface와 Kubernetes management interface 모두 적용되어 한다. 그래서 다른 CNI plug-in들 사이에 interface 역할을 하는 `Multus` CNI plug-in을 사용한다. 
여기서는 Flannel을 Master plug-in인 Kubernetes management interface로 사용할것이고, SR-IOV DPDK plug-in을 fast packet 처리를 위해 사용된다.
Multus를 사용하는 두 가지 다른 방법이 있다. 
- CRD를 이용한다 
	- 네트워크 자원을 CRD로 정의하고 Pod별로 적용한다. (SR-IOV 네트워크 디바이스 plugin 설정 필요)
- 요구된 설정이 있는 CNI 설정 파일을 각 worker 노드에서 가지는 것이다. 
	- 이것은 worker 노드에서 실행되고 있는 모든 pods가 같은 설정파일을 참조할 수 밖에 없으므로 Pod별로 네트워크를 customize할 수 없는 단점이 있지만 여기서는 pod별로 세세하게 설정을 안해도 되므로 두번째를 사용한다. 

Multus 설정 파일은 아래와 같다.
```
$ cat /etc/cni/net.d/10-multus.conf
{
  "name": "multus-demo-network",
  "type": "multus",
  "delegates ": [
    {
      "type": "flannel",
      "masterplug -in": true,
      "delegate ": {
      "isDefaultGateway": true
      }
    },
    {
      "type": "sriov",
      "name": "net1",
      "if0": "enp1s0f1",
      "if0name": "if0",
      "dpdk": {
        "kernel_driver":"ixgbevf",
        "dpdk_driver":"igb_uio",
        "dpdk_tool":"\$RTE_SDK/usertools/dpdk-devbind.py"
      }
    },
    {
      "type": "sriov",
      "name": "net2",
      "if0": "enp1s0f2",
      "if0name ": "if1",
      "dpdk": {
        "kernel_driver":"ixgbevf",
        "dpdk_driver":"igb_uio",
        "dpdk_tool":"\$RTE_SDK/usertools/dpdk-devbind.py"
      }
    }
  ]
}
```
