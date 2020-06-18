+++
title = "Installing Kolla-kubernetes"
description = ""
tags = [
    "Kubernetes",
    "Openstack",
]
date = "2017-10-21"
categories = [
    "Kubernetes",
    "Openstack",
]
highlight = "true"
+++

kolla로 build된 Openstack components를 
Kubernetes상에 전개하는 Kolla-kubernetes를 설치하는 방법을 정리하였다.
  
기존 소스에서 아래를 수정 하였다.
 - All-in-one으로 설치되는 Charts를 Controller/Network/Compute node로 분할하여 Multi-node 로 설치되도록 Open vSwitch관련 Charts를 수정.
 - Openstack Ocata(4.0.0)를 최신 release인 Pike(5.0.2)로 변경 적용.
 - 안정적으로 구동될 수 있도록 기존의 많은 문제점들을 수정.


## git clone

{% highlight js %}
git clone http://github.com/openstack/kolla-ansible
git clone http://github.com/openstack/kolla-kubernetes
{% endhighlight %}
  
  
## configure files

### replace 4.0.0 with 5.0.2

뭐 tools, orchestration 은 안해도 될것 같지만....
{% highlight sh %}
vim kolla-kubernetes/tools/setup_kube_AIO.sh
vim kolla-kubernetes/tools/setup_gate.sh
vim kolla-kubernetes/helm/all_values.yaml
vim kolla-kubernetes/orchestration/roles/kolla-controller/templates/cloud.yaml
{% endhighlight %}

### kolla-kubernetes/helm/all_values

- apply IPs
```
docker_registry: 192.168.125.159:5000
kolla_kubernetes_external_vip: 172.30.1.201
```
- replace all version No. with 5.0.2
```
                    haproxy_image_tag: 5.0.2
```
- api_interface for keepalived-daemonset
```
                        api_interface: eth0
```

### /etc/kolla/globals.yml

```
kolla_internal_vip_address: "172.30.1.201"
docker_registry: "192.168.125.159:5000"
network_interface: "eth0"
api_interface: "{{ network_interface }}"
neutron_external_interface: "eth1"
.
.
kolla_install_type: "binary"
```

### cloud.yaml

```
global:
   kolla:
     all:
       docker_registry: 192.168.125.159:5000
       image_tag: "5.0.2"
       kube_logger: false
       external_vip: "172.30.1.201"
       base_distro: "centos"
       install_type: "binary"
       tunnel_interface: "eth0"
     keystone:
       all:
         admin_port_external: "true"
         dns_name: "172.30.1.201"
       public:
         all:
           port_external: "true"
     rabbitmq:
       all:
         cookie: 67
     glance:
       api:
         all:
           port_external: "true"
     cinder:
       api:
         all:
           port_external: "true"
       volume_lvm:
         all:
           element_name: cinder-volume
         daemonset:
           lvm_backends:
           - '172.30.1.201': 'cinder-volumes'
     ironic:
       conductor:
         daemonset:
           selector_key: "kolla_conductor"
     nova:
       placement_api:
         all:
           port_external: true
       novncproxy:
         all:
           port: 6080
           port_external: true
     openvswitch:
       all:
         add_port: true
         ext_bridge_name: br-ex
         ext_interface_name: eth1
         setup_bridge: true
     horizon:
       all:
         port_external: true
```
  
  
## install

```
sudo pip install -U kolla-ansible/ kolla-kubernetes/
sudo cp -aR /usr/share/kolla-ansible/etc_examples/kolla /etc
sudo cp -aR kolla-kubernetes/etc/kolla-kubernetes /etc
sudo kolla-kubernetes-genpwd
kubectl create namespace kolla
kubectl label kkolla03 kolla_compute=true
kubectl label kkolla02 kolla_network=true
kubectl label kkolla01 kolla_controller=true

sudo mkdir /etc/kolla/config
sudo tee /etc/kolla/config/nova.conf<<EOF
[libvirt]
virt_type=qemu
cpu_mode=none
EOF

ansible-playbook -e ansible_python_interpreter=/usr/bin/python -e @/etc/kolla/globals.yml -e @/etc/kolla/passwords.yml -e CONFIG_DIR=/etc/kolla kolla-kubernetes/ansible/site.yml

kolla-kubernetes/tools/secret-generator.py create
kollakube res create configmap \
    mariadb keystone horizon rabbitmq memcached nova-api nova-conductor \
    nova-scheduler glance-api-haproxy glance-registry-haproxy glance-api \
    glance-registry neutron-server neutron-dhcp-agent neutron-l3-agent \
    neutron-metadata-agent neutron-openvswitch-agent openvswitch-db-server \
    openvswitch-vswitchd nova-libvirt nova-compute nova-consoleauth \
    nova-novncproxy nova-novncproxy-haproxy neutron-server-haproxy \
    nova-api-haproxy cinder-api cinder-api-haproxy cinder-backup \
    cinder-scheduler cinder-volume iscsid tgtd keepalived \
    placement-api placement-api-haproxy

kolla-kubernetes/tools/helm_build_all.sh .
helm install kolla-kubernetes/helm/microservice/keepalived-daemonset --namespace kolla --name keepalived --values ./cloud.yaml
helm install --debug kolla-kubernetes/helm/service/mariadb --namespace kolla --name mariadb --values ./cloud.yaml
helm install --debug kolla-kubernetes/helm/service/rabbitmq --namespace kolla --name rabbitmq --values ./cloud.yaml
helm install --debug kolla-kubernetes/helm/service/memcached --namespace kolla --name memcached --values ./cloud.yaml
helm install --debug kolla-kubernetes/helm/service/keystone --namespace kolla --name keystone --values ./cloud.yaml
helm install --debug kolla-kubernetes/helm/service/glance --namespace kolla --name glance --values ./cloud.yaml
helm install --debug kolla-kubernetes/helm/service/cinder-control --namespace kolla --name cinder-control --values ./cloud.yaml
helm install --debug kolla-kubernetes/helm/service/horizon --namespace kolla --name horizon --values ./cloud.yaml
```
  
  


## network-node 분리
기존 All-in-one에서 Controller, Network, Compute로 분할하여 Multi node로 deploy하기 위해서는
`Open vSwitch` 관련 설정을 변경해야 한다. <br/>
Network node에는 Open vSwitch와 Neutron관련 Components를 deploy하고<br/>
그 중 아래 3개의 components는 Compute node의 OVS설정을 위해서 deploy한다.<br/>
내용은 Network node와 비슷하지만 Compute node에는 br-ex가 적용되면 안되므로 neutron-openvswitch-agent-daemonset의 ml2 설정에서 br-ex를 삭제한다. 뒤에서 설명함.<br/>
```
neutron-openvswitch-agent-daemonset
openvswitch-vswitchd-daemonset
openvswitch-ovsdb-daemonset
```
이들 pod들은 하나의 `deployment`로 deploy하지 않고 node 각각에 적용하도록 변경함
그러기 위해서 
  

1) kolla_network label을 추가함
```
kubectl label node kkolla01 kolla_controller=true -> 원래 있던것
kubectl label node kkolla02 kolla_network=true    -> New
kubectl label node kkolla03 kolla_compute=true    -> 원래 있던것
```
2) Microservice 변경
- Chart 복제

```
cd /root/kolla-bringup/kolla-kubernetes/helm
cp -r microservice/openvswitch-ovsdb-daemonset microservice/openvswitch-ovsdb-network-daemonset
mv microservice/openvswitch-ovsdb-daemonset microservice/openvswitch-ovsdb-compute-daemonset

cp -r microservice/openvswitch-vswitchd-daemonset microservice/openvswitch-vswitchd-network-daemonset
mv microservice/openvswitch-vswitchd-daemonset microservice/openvswitch-vswitchd-compute-daemonset

cp -r microservice/neutron-openvswitch-agent-daemonset microservice/neutron-openvswitch-agent-network-daemonset
mv microservice/neutron-openvswitch-agent-daemonset microservice/neutron-openvswitch-agent-compute-daemonset


```

- Chart 수정

```
vim microservice/openvswitch-ovsdb-network-daemonset/Chart.yaml
-> name: openvswitch-ovsdb-network-daemonset
vim microservice/openvswitch-ovsdb-compute-daemonset/Chart.yaml
-> name: openvswitch-ovsdb-compute-daemonset
vim microservice/openvswitch-vswitchd-network-daemonset/Chart.yaml
-> name: openvswitch-vswitchd-network-daemonset
vim microservice/openvswitch-vswitchd-compute-daemonset/Chart.yaml
-> name: openvswitch-vswitchd-compute-daemonset
vim microservice/neutron-openvswitch-agent-network-daemonset/Chart.yaml
-> name: neutron-openvswitch-agent-network-daemonset
vim microservice/neutron-openvswitch-agent-compute-daemonset/Chart.yaml
-> name: neutron-openvswitch-agent-compute-daemonset
```

- template name 변경

```
mv microservice/openvswitch-ovsdb-network-daemonset/templates/openvswitch-ovsdb-daemonset.yaml microservice/openvswitch-ovsdb-network-daemonset/templates/openvswitch-ovsdb-network-daemonset.yaml
mv microservice/openvswitch-ovsdb-compute-daemonset/templates/openvswitch-ovsdb-daemonset.yaml microservice/openvswitch-ovsdb-compute-daemonset/templates/openvswitch-ovsdb-compute-daemonset.yaml
mv microservice/openvswitch-vswitchd-network-daemonset/templates/openvswitch-vswitchd-daemonset.yaml microservice/openvswitch-vswitchd-network-daemonset/templates/openvswitch-vswitchd-network-daemonset.yaml
mv microservice/openvswitch-vswitchd-compute-daemonset/templates/openvswitch-vswitchd-daemonset.yaml microservice/openvswitch-vswitchd-compute-daemonset/templates/openvswitch-vswitchd-compute-daemonset.yaml
mv microservice/neutron-openvswitch-agent-network-daemonset/templates/openvswitch-agent-daemonset.yaml microservice/neutron-openvswitch-agent-network-daemonset/templates/openvswitch-agent-network-daemonset.yaml
mv microservice/neutron-openvswitch-agent-compute-daemonset/templates/openvswitch-agent-daemonset.yaml microservice/neutron-openvswitch-agent-compute-daemonset/templates/openvswitch-agent-compute-daemonset.yaml
```

3) 기존 패키지 삭제 (삭제 안하면 중복돼서 deploy 안됨)

```
cd /root/kolla-bringup/
rm openvswitch-ovsdb-daemonset-0.7.0-1.tgz
rm openvswitch-vswitchd-daemonset-0.7.0-1.tgz
rm neutron-openvswitch-agent-daemonset-0.7.0-1.tgz

cd /root/kolla-bringup/kolla-kubernetes/helm/service/openvswitch/charts/
rm openvswitch-ovsdb-daemonset-0.7.0-1.tgz
rm openvswitch-vswitchd-daemonset-0.7.0-1.tgz

cd /root/kolla-bringup/kolla-kubernetes/helm/service/neutron/charts/
rm neutron-openvswitch-agent-daemonset-0.7.0-1.tgz
```

4) vim all_values.yaml 수정 <br/>
**NOTE** : type은 deployment의 이름에 붙는것이다. 그래서 같이 주면 쫑남<br/>
**NOTE** : selector_key를 주는 방법은 `kolla-kubernetes-정리.md`의 `nodeSelector` 참고<br/>

```
.
.
.
neutron-openvswitch-agent-network-daemonset:
        type: network
        global:
            kolla:
                all:
                    tunnel_interface: eth0
                neutron:
                    openvswitch_agent_network:
                        daemonset:
                            selector_key: kolla_network
                            logger_configmap_name: neutron-openvswitch-agent-logger

neutron-openvswitch-agent-compute-daemonset:
        type: compute
        global:
            kolla:
                all:
                    tunnel_interface: eth0
                neutron:
                    openvswitch_agent_compute:
                        daemonset:
                            selector_key: kolla_compute
                            logger_configmap_name: neutron-openvswitch-agent-logger

neutron-l3-agent-daemonset:
        type: network
        global:
            kolla:
                all:
                    tunnel_interface: eth0
                neutron:
                    all:
                        dvr: false
                    l3_agent_network:
                        daemonset:
                            selector_key: kolla_network
                            logger_configmap_name: neutron-l3-agent-logger

neutron-dhcp-agent-daemonset:
        type: network
        global:
            kolla:
                all:
                    tunnel_interface: eth0
                neutron:
                    dhcp_agent:
                        daemonset:
                            selector_key: kolla_network
                            logger_configmap_name: neutron-dhcp-agent-logger

neutron-metadata-agent-daemonset:
        type: network
        global:
            kolla:
                neutron:
                    metadata_agent_network:
                        daemonset:
                            selector_key: kolla_network
                            logger_configmap_name: neutron-metadata-agent-logger

openvswitch-ovsdb-network-daemonset:
        type: network
        global:
            kolla:
                openvswitch:
                    db_network:
                        daemonset:
                            selector_key: kolla_network

openvswitch-ovsdb-compute-daemonset:
        type: compute
        global:
            kolla:
                openvswitch:
                    db_compute:
                        daemonset:
                            selector_key: kolla_compute

openvswitch-vswitchd-network-daemonset:
        type: network
        global:
            kolla:
                openvswitch:
                    all:
                        selector_key: kolla_network
                        setup_bridge: true
                        add_port: false
                        ext_bridge_name: br-ex
                        ext_bridge_up: false
                        ext_interface_name: eth1
                    vswitchd_network:                          //이렇게 "_network"를 붙여서 구분해 함. 이유는 아래에 있음
                        daemonset:
                            selector_key: kolla_network

openvswitch-vswitchd-compute-daemonset:
        type: compute
        global:
            kolla:
                openvswitch:
                    all:
                        selector_key: kolla_compute
                        setup_bridge: true
                        add_port: false
                        ext_bridge_name: br-ex
                        ext_bridge_up: false
                        ext_interface_name: eth1
                    vswitchd_compute:                         //이렇게 "_compute"를 붙여서 구분하지 않으면 이전 값을 참조하는지 kolla_network에 설치 된다.
                        daemonset:
                            selector_key: kolla_compute


```

5) dependency.yaml 수정

vim /root/kolla-bringup/kolla-kubernetes/helm/service/openvswitch/requirements.yaml

```
dependencies:
  - name: openvswitch-ovsdb-network-daemonset
    repository: file://../../microservice/openvswitch-ovsdb-network-daemonset
    version: 0.7.0-1
  - name: openvswitch-vswitchd-network-daemonset
    repository: file://../../microservice/openvswitch-vswitchd-network-daemonset
    version: 0.7.0-1
  - name: openvswitch-ovsdb-compute-daemonset
    repository: file://../../microservice/openvswitch-ovsdb-compute-daemonset
    version: 0.7.0-1
  - name: openvswitch-vswitchd-compute-daemonset
    repository: file://../../microservice/openvswitch-vswitchd-compute-daemonset
    version: 0.7.0-1
```

vim /root/kolla-bringup/kolla-kubernetes/helm/service/neutron/requirements.yaml

```
.
.
  - name: neutron-openvswitch-agent-network-daemonset
    repository: file://../../microservice/neutron-openvswitch-agent-network-daemonset
    version: 0.7.0-1
  - name: neutron-openvswitch-agent-compute-daemonset
    repository: file://../../microservice/neutron-openvswitch-agent-compute-daemonset
    version: 0.7.0-1
.
.
```

6) 기존 tgz 삭제

```
rm /root/kolla-bringup/kolla-kubernetes/helm/service/openvswitch/charts/openvswitch-ovsdb-daemonset-0.7.0-1.tgz
rm /root/kolla-bringup/kolla-kubernetes/helm/service/openvswitch/charts/openvswitch-vswitchd-daemonset-0.7.0-1.tgz
rm /root/kolla-bringup/kolla-kubernetes/helm/service/neutron/charts/neutron-openvswitch-agent-daemonset-0.7.0-1.tgz
```




7) compute-node에 br-ex삭제
  
vim kolla-kubernetes/helm/microservice/neutron-openvswitch-agent-compute-daemonset/templates/openvswitch-agent-compute-daemonset.yaml
update-config의 command 부분 제일 마지막에 아래 삽입
```
              sed -i "/br-ex/d" /srv/pod-main-config/ml2_conf.ini;
```
vim kolla-kubernetes/helm/all_values.yaml 에 openvswitch-vswitchd-compute-daemonset: 에서 아래로 수정
```
                        selector_key: kolla_compute
                        setup_bridge: false
                        add_port: false
                        ext_bridge_up: false
```

8) remove the below openvswitch from cloud.yaml
```
     openvswitch: 
       all: 
         add_port: true 
         ext_bridge_name: br-ex 
         ext_interface_name: eth1 
         setup_bridge: true 
```

9) rebuild
kolla-kubernetes/tools/helm_build_all.sh .

10) install

```
helm install --debug kolla-kubernetes/helm/service/openvswitch --namespace kolla --name openvswitch --values ./cloud.yaml
helm install --debug kolla-kubernetes/helm/service/neutron --namespace kolla --name neutron --values ./cloud.yaml
```
  
## nova-control 수정 

### 걍 fiexd port로 추가

vim kolla-kubernetes/helm/microservice/nova-api-create-simple-cell-job/templates/nova-api-create-cell.yaml
```
{% raw %}
{{- $keystonePort := include "kolla_val_get_str" (dict "key" "port" "searchPath" $keystoneSearchPath "Values" .Values ) }}
   ->
{{- $keystonePort := include "kolla_val_get_str" (dict "key" "port" "searchPath" $keystoneSearchPath "Values" .Values ) | default "5000" }}
{% endraw %}
```

### dependency 재 설정

vim kolla-kubernetes/helm/service/nova-control/values.yaml <br/>
- cell0 dependency 에서 nova-api 삭제<br/>
- manage_db에 nova-cell0-create-db job 추가 (nova-manage 명령어가 오래 걸려서 그런지 이거 추가 안해도 구동은 됐음)
```
                manage_db:
                    job:
                        dependencies:
                            jobs:
                            - nova-api-create-db
                            - nova-cell0-create-db
                            service:
                            - mariadb
```

### db 초기화(?) 절차 수정 - 아래 두가지 중에 하나 선택

https://docs.openstack.org/nova/pike/install/controller-install-obs.html 참조해서 만듦
  
1) pod를 수정하는 방법 - 공통으로 쓰는 _common_manage_db_job.yaml에 직접 수정하면 안되고 아래처럼 파라미터로 받아서 처리하게 해야 함
 
vim kolla-kubernetes/helm/kolla-common/templates/_common_manage_db_job.yaml
 
아래 추가
```
{% raw %}
{{- if .podTypeCommand }}
          command: ["bin/bash", "-c", {{ .podTypeCommand }}]
{{- end }}
{% endraw %}
```

vim kolla-kubernetes/helm/microservice/nova-api-manage-db-job/templates/nova-api-manage-db.yaml
 
아래 추가
```
{% raw %}
{{- $podTypeCommand := "sudo -E kolla_set_configs; mkdir -p /var/log/kolla/nova; touch /var/log/kolla/nova/nova-manage.log; chmod 777 /var/log/kolla/nova/nova-manage.log; nova-manage api_db sync; nova-manage cell_v2 map_cell0; nova-manage cell_v2 create_cell --name=cell1 --verbose; nova-manage db sync" }}

{{- with $env := dict "resourceName" $resourceName "serviceName" $serviceName "podTypeCommand" $podTypeCommand "podTypeBootstrap" $podTypeBootstrap "imageFull" $imageFull "Values" .Values "Release" .Release "searchPath" $searchPath }}
{% endraw %}
```
<br/>

> nova-manage db online_data_migrations 이 제일 마지막에 있는데 이것도 compute-control 이후에 돌려볼것 돌려봐서 무리 없으면 kolla_build에 넣을것. 이건 새로 설치할때가 아니라 업그레이드할때 필요한것임(이건 적용 안했음)
  
2) 위의 1)로 해도 되고 아님 아래처럼 kolla image를 직접 변경할 수도 있다.
<br/>
vim /root/kolla/docker/nova/nova-api/extend_start.sh
  
```
nova-manage api_db sync;
nova-manage cell_v2 map_cell0;
nova-manage cell_v2 create_cell --name=cell1 --verbose;
nova-manage db sync;
```
<br/>

> 적용한 후 `kolla build` 재 실행
<br/>

### 설치

```
helm install --debug kolla-kubernetes/helm/service/nova-control --namespace kolla --name nova-control --values ./cloud.yaml
helm install --debug kolla-kubernetes/helm/service/nova-compute --namespace kolla --name nova-compute --values ./cloud.yaml
```
### 구조

![topoloty](/images/kolla-k8s.png)
