+++
title = "Tuning kernel for kubernetes"
description = ""
tags = [
    "Kernal parameter",
    "kubernetes",
]
date = "2018-12-28"
categories = [
    "Kubernetes",
]
highlight = "true"
+++

Kubernetes의 Pod/container가 cgroup으로 isolation된다고 하더라도 host의 conntrack table을 공유할 수 밖에 없으므로 
동시 접속자 수가 많고 짧은 응답시간이 요구되는 시스템의 경우 관련 파라미터들의 튜닝이 필요하다.
또한 성능을 위해서 ingress-controller를 hostNetwork로 설정하고 별도의 노드로 구성하였다면 다양한 커널 파라미터들을 튜닝하여 성능 개선을 할 수 있다.
  
# Netfilter modules
## nf_conntrack
`/proc/sys/net/netfilter/nf_conntrack_max`는 NAT/SNAT(또는 IP당 연결 갯수 제한을 위해) 연결을 tracking하는 
table의 최대 연결 갯수를 정하는 값인데, Load test나 많은 연결을 요할때 병목이 될 수 있다.  
확인하는 방법은 /proc/sys/net/netfilter/nf_conntrack_count(현재 커넥션 수)가 nf_conntrack_max값이 되고,  
시스템 로그에 "nf_conntrack: table full, dropping packet"라고 찍힌다면 해당 값에 병목이 발생한 것이다.  
  
  
그런데 kube-proxy가 nf_conntrack_max를 포함한 아래 4가지 nf_conntrack관련 parameters를 부팅시 초기화 한다.
```
/proc/sys/net/netfilter/nf_conntrack_max
/proc/sys/net/netfilter/nf_conntrack_tcp_timeout_established
/proc/sys/net/netfilter/nf_conntrack_tcp_timeout_close_wait
/sys/module/nf_conntrack/parameters/hashsize
```
kube-proxy가 nf_conntrack_max값을 정하는 방법은 [소스 참조](https://github.com/kubernetes/kubernetes/blob/8de1569ddae62e8fab559fe6bd210a5d6100a277/cmd/kube-proxy/app/server.go#L657)  
즉, 아래와 같이 노드에 할당된 core수를 참조하여 설정한다.
```
if((host의 core 갯수 * 32768) > 131072)
  nf_conntrack_max = core 수 * 32768;
else
  nf_conntrack_max = 131072;
```
nf_conntrack_max값을 수정하는 방법은 3가지 이다.
1. host의 core수를 늘린다.
2. kube-proxy의 옵션값을 조정한다.(주의: 옵션은 전체 노드에 적용되므로 신중히 판단해야 함)
```
    --conntrack-max-per-core
    --conntrack-min
```
3. crontab으로 주기적으로 재 설정한다.
```
crontab -l | { cat; echo  "* * * * * COMM=\$(cat /proc/sys/net/netfilter/nf_conntrack_max); if [ \${COMM} -ne $최대값 ]; then /usr/sbin/sysctl -w net.netfilter.nf_conntrack_max=$최대값; fi"; } | crontab -
```

## net.
- net.core.netdev_max_backlog
	- 각 네트워크 장치 별로 커널이 처리하도록 쌓아두는 queue의 크기, trade-off 관계가 메모리 사용량 밖에 없으므로, 적당히 증가시켜두는 것도 괜찮음.
- net.core.somaxconn
	- listen()으로 바인딩 된 서버 소켓에서 accept()를 기다리는 소켓 개수에 관련된 커널 파라미터.
- net.ipv4.tcp_max_syn_backlog
	- SYN_RECEIVED 상태의 소켓(connection incompleted)을 위한 queue 이다.

# TIME_WAIT socket reuse

웹서버가 다른 서버에 질의하는 경우, 서버는 또 다른 서버의 클라이언트가 되므로 TIME_WAIT 상태의 소켓 개수가 성능에 영향을 미칠 수 있다.  
이것을 사용하면 socket의 활용도를 높일 수 있지만, TIME_WAIT 상태는 gracefully shutdown을 위해 필요하므로, 무조건 없애는것은 문제가 될 수 있다.  
특히, NAT와 유사한 환경에서는 문제가 발생할 수 있다.  

## 방법은 아래 4가지 이다.

> 참고: https://meetup.toast.com/posts/55

### 1. 애플리케이션에서 connection pool을 사용
app에서 별도 개발 필요함
### 2. TW_REUSE 옵션을 사용
사용할 수 있는 local port 수가 모자라면, 현재 TIME_WAIT 상태의 소켓 중 프로토콜상 사용해도 무방해 보이는 소켓을 재사용.
```
      ipv4.tcp_timestamps="1"
      net.ipv4.tcp_tw_reuse="1"
```
유의할 점은, TW_REUSE 옵션은 통신을 하는 양측 모두 TCP timestamp 옵션이 설정되어 있어야 활성화된다

### 3. TW_RECYCLE 옵션을 사용
좀 과격한 방법으로 TIMEWAIT 상태에 머무르는 시간을 변경하여 TIMEWAIT 상태의 소켓 수를 줄여버림.
```
      ipv4.tcp_timestamps="1"
      net.ipv4.tcp_tw_reuse="1"
      net.ipv4.tcp_tw_recycle="1"
```
유의점: 클라이언트가 NAT 환경인 경우 일부 클라이언트로 부터의 SYN 패킷이 유실될 수 있기 때문에 TW_RECYCLE 사용시 일부 TCP 연결 요청은 실패하게 됨.

### 4. Socket linger 옵션을 굉장히 짧은 시간을 매개변수로 주어 사용
가장 과격한 방법으로, FIN 대신 RST를 보내게 유도하며, 이 때 소켓을 파괴하여 소켓이 TIME_WAIT 상태에 머무르지 않게 함.

# Openfile descripter
/etc/security/limits.conf 아래 값 추가하여 soft/hard 모두 늘려준다.
```
*               soft    nofile            [적당한 값]
*               hard    nofile            [적당한 값]
```
아래 명령어로 설정값 확인
```
ulimit -a
ulimit -Sa
ulimit -Ha
```
