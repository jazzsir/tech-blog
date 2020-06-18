+++
title = "Using SR-IOV on OpenStack"
description = ""
tags = [
    "DPDK",
    "SR-IOV",
]
date = "2016-03-05"
categories = [
    "Data plane acceleration",
]
highlight = "true"
+++
## BIOS 설정

Enable Intel VT-d in BIOS

## 네트워크 설정

- SR-IOV 관련 인터페이스 ONBOOT를 yes로 설정

```
/etc/sysconfig/network-scripts/ifcfg-ens1f0
TYPE=Ethernet
BOOTPROTO=none
NAME=ens1f0
DEVICE=ens1f0
ONBOOT=yes
NM_CONTROLLED=no
USERCTL=no

/etc/sysconfig/network-scripts/ifcfg-ens4f0
TYPE=Ethernet
BOOTPROTO=none
NAME=ens4f0
DEVICE=ens4f0
ONBOOT=yes
NM_CONTROLLED=no
USERCTL=no

[root@overcloud-compute-3 network-scripts]# cat ifcfg-ens1f1
TYPE=Ethernet
BOOTPROTO=none
NAME=ens1f1
DEVICE=ens1f1
ONBOOT=yes
NM_CONTROLLED=no
USERCTL=no

[root@overcloud-compute-3 network-scripts]# cat ifcfg-ens4f1
TYPE=Ethernet
BOOTPROTO=none
NAME=ens4f1
DEVICE=ens4f1
ONBOOT=yes
NM_CONTROLLED=no
USERCTL=no
```


- Compute Node PCI Vendor/Product ID 조회

```
[root@overcloud-compute-3 ~]# ethtool -i ens1f1
driver: ixgbe
version: 4.0.1-k-rh7.1
firmware-version: 0x18f60001
bus-info: 0000:04:00.1
supports-statistics: yes
supports-test: yes
supports-eeprom-access: yes
supports-register-dump: yes
supports-priv-flags: no

[root@overcloud-compute-3 ~]# ethtool -i ens4f1
driver: ixgbe
version: 4.0.1-k-rh7.1
firmware-version: 0x8000028d
bus-info: 0000:21:00.1
supports-statistics: yes
supports-test: yes
supports-eeprom-access: yes
supports-register-dump: yes
supports-priv-flags: no

[root@overcloud-compute-3 ~]# lspci -nn | grep Eth
04:00.0 Ethernet controller [0200]: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection [8086:10fb] (rev 01)
04:00.1 Ethernet controller [0200]: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection [8086:10fb] (rev 01)
21:00.0 Ethernet controller [0200]: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection [8086:10fb] (rev 01)
21:00.1 Ethernet controller [0200]: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection [8086:10fb] (rev 01)
```

- SR-IOV VF 생성 설정

```
[root@overcloud-compute-3 ~]# modinfo ixgbe
filename:       /lib/modules/3.10.0-229.14.1.el7.x86_64/kernel/drivers/net/ethernet/intel/ixgbe/ixgbe.ko
version:        4.0.1-k-rh7.1
license:        GPL
description:    Intel(R) 10 Gigabit PCI Express Network Driver
author:         Intel Corporation, <linux.nics@intel.com>
rhelversion:    7.1
srcversion:     D5B70A3E5E8A45DA6B20C73
alias:          pci:v00008086d000015ABsv*sd*bc*sc*i*
alias:          pci:v00008086d000015AAsv*sd*bc*sc*i*
alias:          pci:v00008086d00001563sv*sd*bc*sc*i*
alias:          pci:v00008086d00001560sv*sd*bc*sc*i*
alias:          pci:v00008086d0000154Asv*sd*bc*sc*i*
alias:          pci:v00008086d00001557sv*sd*bc*sc*i*
alias:          pci:v00008086d00001558sv*sd*bc*sc*i*
alias:          pci:v00008086d0000154Fsv*sd*bc*sc*i*
alias:          pci:v00008086d0000154Dsv*sd*bc*sc*i*
alias:          pci:v00008086d00001528sv*sd*bc*sc*i*
alias:          pci:v00008086d000010F8sv*sd*bc*sc*i*
alias:          pci:v00008086d0000151Csv*sd*bc*sc*i*
alias:          pci:v00008086d00001529sv*sd*bc*sc*i*
alias:          pci:v00008086d0000152Asv*sd*bc*sc*i*
alias:          pci:v00008086d000010F9sv*sd*bc*sc*i*
alias:          pci:v00008086d00001514sv*sd*bc*sc*i*
alias:          pci:v00008086d00001507sv*sd*bc*sc*i*
alias:          pci:v00008086d000010FBsv*sd*bc*sc*i*
alias:          pci:v00008086d00001517sv*sd*bc*sc*i*
alias:          pci:v00008086d000010FCsv*sd*bc*sc*i*
alias:          pci:v00008086d000010F7sv*sd*bc*sc*i*
alias:          pci:v00008086d00001508sv*sd*bc*sc*i*
alias:          pci:v00008086d000010DBsv*sd*bc*sc*i*
alias:          pci:v00008086d000010F4sv*sd*bc*sc*i*
alias:          pci:v00008086d000010E1sv*sd*bc*sc*i*
alias:          pci:v00008086d000010F1sv*sd*bc*sc*i*
alias:          pci:v00008086d000010ECsv*sd*bc*sc*i*
alias:          pci:v00008086d000010DDsv*sd*bc*sc*i*
alias:          pci:v00008086d0000150Bsv*sd*bc*sc*i*
alias:          pci:v00008086d000010C8sv*sd*bc*sc*i*
alias:          pci:v00008086d000010C7sv*sd*bc*sc*i*
alias:          pci:v00008086d000010C6sv*sd*bc*sc*i*
alias:          pci:v00008086d000010B6sv*sd*bc*sc*i*
depends:        mdio,ptp,dca
intree:         Y
vermagic:       3.10.0-229.14.1.el7.x86_64 SMP mod_unload modversions
signer:         Red Hat Enterprise Linux kernel signing key
sig_key:        44:02:00:8F:2B:3F:DA:1F:6C:1C:64:BA:2C:10:BF:0C:FE:EC:AB:B7
sig_hashalgo:   sha256
parm:           max_vfs:Maximum number of virtual functions to allocate per physical function - default is zero and maximum value is 63 (uint)
parm:           allow_unsupported_sfp:Allow unsupported and untested SFP+ modules on 82599-based adapters (uint)
parm:           debug:Debug level (0=none,...,16=all) (int)
```

```
Edit /etc/modprobe.d/ixgbe.conf

options ixgbe max_vfs=16
```

```
Edit /etc/sysconfig/grub

GRUB_TIMEOUT=5
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="no_timer_check crashkernel=auto console=ttyS0,115200n8 console=tty0 net.ifnames=1 rhgb quiet intel_iommu=on ixgbe.max_vfs=16"
GRUB_DISABLE_RECOVERY="true"
GRUB_PRELOAD_MODULES=lvm
```

```
Edit /etc/grub2.cfg

...
### BEGIN /etc/grub.d/10_linux ###
menuentry 'Red Hat Enterprise Linux Server 7.1 (Maipo), with Linux 3.10.0-229.14.1.el7.x86_64' --class fedora --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-229.14.1.el7.x86_64-advanced-fa2ae704-b7dc-406b-8747-e6d6d59b0b39' {
        load_video
        set gfxpayload=keep
        insmod gzio
        insmod part_msdos
        insmod xfs
        set root='hd0,msdos2'
        if [ x$feature_platform_search_hint = xy ]; then
          search --no-floppy --fs-uuid --set=root --hint-bios=hd0,msdos2 --hint-efi=hd0,msdos2 --hint-baremetal=ahci0,msdos2  fa2ae704-b7dc-406b-8747-e6d6d59b0b39
        else
          search --no-floppy --fs-uuid --set=root fa2ae704-b7dc-406b-8747-e6d6d59b0b39
        fi
        linux16 /boot/vmlinuz-3.10.0-229.14.1.el7.x86_64 root=UUID=fa2ae704-b7dc-406b-8747-e6d6d59b0b39 ro no_timer_check crashkernel=auto console=ttyS0,115200n8 console=tty0 net.ifnames=1 rhgb quiet intel_iommu=on ixgbe.max_vfs=16
        initrd16 /boot/initramfs-3.10.0-229.14.1.el7.x86_64.img
}
menuentry 'Red Hat Enterprise Linux Server 7.1 (Maipo), with Linux 0-rescue-997aadf25d384ab2be3c52219f65056a' --class fedora --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-0-rescue-997aadf25d384ab2be3c52219f65056a-advanced-fa2ae704-b7dc-406b-8747-e6d6d59b0b39' {
        load_video
        insmod gzio
        insmod part_msdos
        insmod xfs
        set root='hd0,msdos2'
        if [ x$feature_platform_search_hint = xy ]; then
          search --no-floppy --fs-uuid --set=root --hint-bios=hd0,msdos2 --hint-efi=hd0,msdos2 --hint-baremetal=ahci0,msdos2  fa2ae704-b7dc-406b-8747-e6d6d59b0b39
        else
          search --no-floppy --fs-uuid --set=root fa2ae704-b7dc-406b-8747-e6d6d59b0b39
        fi
        linux16 /boot/vmlinuz-0-rescue-997aadf25d384ab2be3c52219f65056a root=UUID=fa2ae704-b7dc-406b-8747-e6d6d59b0b39 ro no_timer_check crashkernel=auto console=ttyS0,115200n8 console=tty0 net.ifnames=1 rhgb quiet intel_iommu=on ixgbe.max_vfs=16
        initrd16 /boot/initramfs-0-rescue-997aadf25d384ab2be3c52219f65056a.img
}
...
```

```
$ grub2-mkconfig -o /boot/grub2/grub.cfg
$ reboot
$ modprobe -r ixgbe
$ modprobe ixgbe max_vfs=16
```

- VF 생성 확인

```
[root@overcloud-compute-3 ~]# lspci -nn | grep Eth
03:00.0 Ethernet controller [0200]: Broadcom Corporation NetXtreme BCM5719 Gigabit Ethernet PCIe [14e4:1657] (rev 01)
03:00.1 Ethernet controller [0200]: Broadcom Corporation NetXtreme BCM5719 Gigabit Ethernet PCIe [14e4:1657] (rev 01)
03:00.2 Ethernet controller [0200]: Broadcom Corporation NetXtreme BCM5719 Gigabit Ethernet PCIe [14e4:1657] (rev 01)
03:00.3 Ethernet controller [0200]: Broadcom Corporation NetXtreme BCM5719 Gigabit Ethernet PCIe [14e4:1657] (rev 01)
04:00.0 Ethernet controller [0200]: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection [8086:10fb] (rev 01)
04:00.1 Ethernet controller [0200]: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection [8086:10fb] (rev 01)
04:10.0 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:10.1 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:10.2 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:10.3 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:10.4 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:10.5 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:10.6 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:10.7 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:11.0 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
04:11.1 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
...
```

- Edit /etc/nova/nova.conf

Intel NIC의 경우, 10G (2 Port) 에서 짝수 번호는 첫 번째 포트, 홀수는 두 번째 포트를 의미함
여기서, 첫 번째 포트는 서비스 네트워크 용도로 사용하고 두 번째 포트는 Interwork (Hybrid) 네트워크 용도로 사용

```
...
pci_passthrough_whitelist={"address":"*:04:*.0", "physical_network":"01service"}
pci_passthrough_whitelist={"address":"*:04:*.2", "physical_network":"01service"}
pci_passthrough_whitelist={"address":"*:04:*.4", "physical_network":"01service"}
pci_passthrough_whitelist={"address":"*:04:*.6", "physical_network":"01service"}

pci_passthrough_whitelist={"address":"*:04:*.1", "physical_network":"02interwork"}
pci_passthrough_whitelist={"address":"*:04:*.3", "physical_network":"02interwork"}
pci_passthrough_whitelist={"address":"*:04:*.5", "physical_network":"02interwork"}
pci_passthrough_whitelist={"address":"*:04:*.7", "physical_network":"02interwork"}
...

[root@compute ~]# systemctl restart openstack-nova-compute
```

- Neutron Agent 설치 및 설정

```
# ##Neutron SR-IOV Agent 설치
# yum install openstack-neutron-sriov-nic-agent.noarc
# systemctl enable neutron-sriov-nic-agent.service
```

```
Edit /etc/neutron/plugins/ml2/ml2_conf_sriov.ini

[ml2_sriov]
supported_pci_vendor_devs = 8086:10ed
agent_required = True

[sriov_nic]
physical_device_mappings = 02interwork:ens1f0,02interwork:ens1f1
exclude_devices =
```

```
Edit /usr/lib/systemd/system/neutron-sriov-nic-agent.service

위에서 설정한 /etc/neutron/plugins/ml2/ml2_conf_sriov.ini 파일 추가

[Unit]
Description=OpenStack Neutron SR-IOV NIC Agent
After=syslog.target network.target

[Service]
Type=simple
User=neutron
ExecStart=/usr/bin/neutron-sriov-nic-agent --config-file /usr/share/neutron/neutron-dist.conf --config-file /etc/neutron/neutron.conf --config-dir /etc/neutron/conf.d/common --config-dir /etc/neutron/conf.d/neutron-sriov-nic-agent --log-file /var/log/neutron/sriov-nic-agent.log --config-file /etc/neutron/plugins/ml2/ml2_conf_sriov.ini
PrivateTmp=false
KillMode=process

[Install]
WantedBy=multi-user.target
```

```
Edit /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini

[ovs]
integration_bridge = br-int
enable_tunneling=False

[agent]
polling_interval = 2
l2_population = False
arp_responder = False
enable_distributed_routing = False

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

```
$ openstack-service restart neutron
```
