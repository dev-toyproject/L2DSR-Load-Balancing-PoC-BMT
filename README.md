## 📝 프로젝트 소개
 
![image](https://github.com/akgkfk3/L2DSR-Load-Balancing-PoC-BMT/assets/55624470/24145d32-9614-42b1-8748-6176f62aa336)

> 기존 Proxy 방식의 로드밸런서는 HTTP 요청과 응답을 모두 처리하기 때문에 높은 부하 상황에서 성능이 감소할 수 있습니다.
>
> 따라서, LB 서버의 부하를 줄이기 위해 웹 서버에서 직접 클라이언트에게 응답하는 DSR (Direct Server Return) 방식의 로드밸런서를 구축하여 Proxy 방식 대비 어느 정도의 성능 향상이 이루어지는 지 알고자 BMT를 진행하게 되었습니다.
> 
> 참고로 DSR 방식은 크게 L2DSR (Mac 주소 변조), L3DSR (Tunnel or DSCP)로 구분되지만 해당 프로젝트에서는 L2DSR 방식을 사용합니다!  <br/>
> 
> 프로젝트 인원 : 1명
>  
> 프로젝트 기간 : 2022.07.22 ~ 2022.08.01

<br/>

## 🛠 사용 기술

<img src="https://img.shields.io/badge/linux-FCC624?style=for-the-badge&logo=linux&logoColor=black"> <img src="https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=Ansible&logoColor=black"> <img src="https://img.shields.io/badge/HAProxy-0061FF?style=for-the-badge&logoColor=black"> <img src="https://img.shields.io/badge/Linux Virtual Server-0061FF?style=for-the-badge&logoColor=black"> <img src="https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=Nginx&logoColor=black">


<br/>

## 🔎 프로젝트 구상도

![image](https://github.com/akgkfk3/L2DSR-Load-Balancing-PoC-BMT/assets/55624470/559da4c4-1f77-405b-9c85-15878500bfec)


<br/>

## 🌏 환경 구축

<details>
<summary>Client VM 구축</summary>

<br/>

<b>1. 벤치마크 테스트 툴 설치 (Rocky Linux 8) </b>

```
dnf install -y httpd-tools 
```

<b>2. TCP 커널 튜닝 </b>

- Setting sysctl.conf

```
cat <<EOF > /etc/sysctl.conf

### TCP
net.core.somaxconn = 250000
net.ipv4.tcp_max_orphans = 15000
net.ipv4.tcp_max_tw_buckets = 700000
net.ipv4.tcp_fin_timeout = 120
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 5
net.ipv4.tcp_max_syn_backlog = 1000000
net.ipv4.tcp_keepalive_time = 30
net.ipv4.tcp_keepalive_intvl = 5
net.ipv4.tcp_keepalive_probes = 10
net.ipv4.tcp_window_scaling = 1
net.core.netdev_max_backlog = 1000000
net.netfilter.nf_conntrack_max = 700000

### Socket Buffer Size & Memory
### Unit = byte / only net.ipv4.tcp_mem's Unit = Page (4096 byte)
net.core.rmem_default=253952
net.core.wmem_default=253952
net.core.rmem_max=16777216
net.core.wmem_max=16777216
net.ipv4.tcp_rmem=4096 16384 4194304
net.ipv4.tcp_wmem=8192 16384 4194304
net.ipv4.tcp_mem = 185688 247584 6291456

### file-descripter (Linux에서는 TCP 소켓 또한 파일로 취급)
fs.file-max = 2000000

### ETC
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 0
net.ipv4.ip_local_port_range = 1024 65535
vm.swappiness = 100
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

EOF
```

- Apply sysctl.conf

```
sysctl -p --system
```

</details>

<details>
<summary>Web VM 구축</summary>

<br/>

<b>1. Nginx 웹 서버 구축 (Rocky Linux 8) </b>

- Install Nginx
```
dnf install -y nginx
```

- Setting Nginx Configuration

```
cat <<EOF > /etc/nginx/nginx.conf
user nginx;
worker_processes 2;
worker_rlimit_nofile 1000000;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
	worker_connections 200000;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    client_body_timeout 2m;
    client_header_timeout 2m;
    send_timeout 2.5m;
    resolver_timeout 2m;
    keepalive_timeout   2m;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80 default_server backlog=300000;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

    # https 설정은 생략
}
EOF
```

- Restart Nginx Service

```
systemctl daemon-reload
systemctl restart nginx
```

<b>2. TCP 커널 튜닝 </b>

- Setting sysctl.conf

```
cat <<EOF > /etc/sysctl.conf

### TCP
net.core.somaxconn = 500000
net.ipv4.tcp_max_orphans = 30000
net.ipv4.tcp_max_tw_buckets = 1500000
net.ipv4.tcp_fin_timeout = 120
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 5
net.ipv4.tcp_max_syn_backlog = 1000000
net.ipv4.tcp_keepalive_time = 30
net.ipv4.tcp_keepalive_intvl = 5
net.ipv4.tcp_keepalive_probes = 10
net.ipv4.tcp_window_scaling = 1
net.core.netdev_max_backlog = 1000000
net.core.netdev_budget = 100000
net.core.netdev_weight = 10000
net.core.netdev_budget_usecs = 5000
net.netfilter.nf_conntrack_max = 700000

### Socket Buffer Size & Memory
### Unit = byte / only net.ipv4.tcp_mem's Unit = Page (4096 byte)
net.core.rmem_default=253952
net.core.wmem_default=253952
net.core.rmem_max=16777216
net.core.wmem_max=16777216
net.ipv4.tcp_rmem=4096 16384 4194304
net.ipv4.tcp_wmem=8192 16384 4194304
net.ipv4.tcp_mem = 185688 247584 6291456

### file-descripter (Linux에서는 TCP 소켓 또한 파일로 취급)
fs.file-max = 2000000

### ETC
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 0
net.ipv4.ip_local_port_range = 1024 65535
vm.swappiness = 100
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

EOF
```

- Apply sysctl.conf

```
sysctl -p --system
```

<b>3. (L2DSR) VIP 설정 </b>

- Setting sysctl.conf

```
ifconfig eth0:0 192.168.16.100/32			// VIP 추가

cat <<EOF >> /etc/sysctl.conf				// Disable ARP (VIP)
### DSR
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.default.arp_ignore = 1
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.eth0.arp_ignore = 1
net.ipv4.conf.eth0.arp_announce = 2
EOF
```

- Apply sysctl.conf

```
sysctl -p --system
```

</details>

<details>
<summary>HAProxy, LVS 로드밸런서 구축</summary>

<br/>

<b>1. HAProxy 로드밸런서 구축 (Rocky Linux 8) </b>

- Install Build Package

```
dnf install -y gcc openssl-devel make pcre-devel readline-devel systemd-devel
```

- Install Lua 5.3

```
wget https://www.lua.org/ftp/lua-5.3.5.tar.gz --no-check-certificate
tar xvf lua-5.3.5.tar.gz
cd ~/lua-5.3.5
make INSTALL_TOP=/opt/lua-5.3.5 linux install
```

- Install HAProxy 2.2.25

```
wget http://www.haproxy.org/download/2.2/src/haproxy-2.2.25.tar.gz
tar xvf haproxy-2.2.25.tar.gz
cd ~/haproxy-2.2.25
make clean
make -j $(nproc) TARGET=linux-glibc USE_OPENSSL=1 USE_ZLIB=1 USE_LUA=1 USE_PCRE=1 USE_SYSTEMD=1 \
LUA_INC=/opt/lua-5.3.5/include LUA_LIB=/opt/lua-5.3.5/lib 
sudo make install
```

- (Optional) Self-Signed Certificate for HTTPS 

```
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr \
-subj "/C=KR/ST=GeongGi/L=PhanGyo/O=Test/OU=CloudDev/CN=192.168.16.100"
openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt
openssl x509 -text -in server.crt
cat server.crt server.key > server.pem
```

- Setting HAProxy Config

```
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    daemon
      maxconn 500000
      ulimit-n 1048576
      nbthread 4			// 코어 개수
      cpu-map 1/1-4 0-23		// 코어 매핑
    tune.bufsize 16384
    tune.maxrewrite 1024
    #tune.ssl.cachesize 100000
    #tune.ssl.lifetime 600
    pidfile     /var/run/haproxy/lbg_thread.pid
    #stats socket /var/lib/haproxy/lbg_thead4
    tune.ssl.default-dh-param 2048

#---------------------------------------------------------------------
# Defaults settings
#---------------------------------------------------------------------
defaults
    backlog                 15000
    maxconn                 10000
    retries                 3
    mode                    http
    option redispatch
    option dontlognull
    timeout connect 210s
    timeout client 200s
    timeout server 200s
    timeout http-request 180s
    timeout http-keep-alive 180s

#---------------------------------------------------------------------
# Frontend settings
#---------------------------------------------------------------------
frontend haproxy_bmt
    bind 192.168.16.100:80
    bind 192.168.16.100:443 ssl crt /root/haproxy/certs/server.pem
    log 127.0.0.1:514 local1
    rate-limit sessions 120000
    default_backend             web

#---------------------------------------------------------------------
# Backend settings
#---------------------------------------------------------------------
backend web
    balance     roundrobin
    server web01 172.16.0.101:80 check
    server web02 172.16.0.102:80 check
    server web03 172.16.0.103:80 check
    server web04 172.16.0.104:80 check
    server web05 172.16.0.105:80 check
    server web06 172.16.0.106:80 check
    server web07 172.16.0.107:80 check
    server web08 172.16.0.108:80 check
    server web09 172.16.0.109:80 check
    server web10 172.16.0.110:80 check
    server web11 172.16.0.111:80 check
    server web12 172.16.0.112:80 check
    server web13 172.16.0.113:80 check
    server web14 172.16.0.114:80 check
    server web15 172.16.0.115:80 check
    server web16 172.16.0.116:80 check
    server web17 172.16.0.117:80 check
    server web18 172.16.0.118:80 check
    server web19 172.16.0.119:80 check
    server web20 172.16.0.120:80 check
    server web21 172.16.0.121:80 check
    server web22 172.16.0.122:80 check
    server web23 172.16.0.123:80 check
    server web24 172.16.0.124:80 check
    server web25 172.16.0.125:80 check
```

<b>2. LVS 로드밸런서 구축 (Rocky Linux 8) </b>

- Install Linux Virtual Server (1.27v)

```
dnf install -y ipvsadm
```

- Start LVS

```
ipvsadm -A -t 192.168.16.100:80 -s rr
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.101:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.102:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.103:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.104:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.105:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.106:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.107:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.108:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.109:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.110:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.111:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.112:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.113:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.114:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.115:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.116:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.117:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.118:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.119:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.120:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.121:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.122:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.123:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.124:80 -g -x 35000 -y 35000
ipvsadm -a -t 192.168.16.100:80 -r 172.16.0.125:80 -g -x 35000 -y 35000
```

<b>3. TCP 커널 튜닝 </b>

- Setting sysctl.conf

```
cat <<EOF > /etc/sysctl.conf

### TCP
net.core.somaxconn = 500000
net.ipv4.tcp_max_orphans = 30000
net.ipv4.tcp_max_tw_buckets = 1500000
net.ipv4.tcp_fin_timeout = 120
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 5
net.ipv4.tcp_max_syn_backlog = 1000000
net.ipv4.tcp_keepalive_time = 30
net.ipv4.tcp_keepalive_intvl = 5
net.ipv4.tcp_keepalive_probes = 10
net.ipv4.tcp_window_scaling = 1
net.core.netdev_max_backlog = 1000000
net.core.netdev_budget = 100000
net.core.netdev_weight = 10000
net.core.netdev_budget_usecs = 5000
net.netfilter.nf_conntrack_max = 700000

### Socket Buffer Size & Memory
### Unit = byte / only net.ipv4.tcp_mem's Unit = Page (4096 byte)
net.core.rmem_default=253952
net.core.wmem_default=253952
net.core.rmem_max=16777216
net.core.wmem_max=16777216
net.ipv4.tcp_rmem=4096 16384 4194304
net.ipv4.tcp_wmem=8192 16384 4194304
net.ipv4.tcp_mem = 185688 247584 6291456

### file-descripter (Linux에서는 TCP 소켓 또한 파일로 취급)
fs.file-max = 2000000

### ETC
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 0
net.ipv4.ip_local_port_range = 1024 65535
vm.swappiness = 100
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

EOF
```


</details>

<details>
<summary>(옵션) 코어 격리 및 IRQ 설정</summary>

<br/>

<b>[개요]</b>

- 기본적으로 프로세스(스레드)는 운영체제에게 CPU를 할당 받아서 자신의 문맥 (Context)를 실행한다.

- 하지만 모든 CPU 코어는 공유하기 때문에 프로세스(스레드) 간의 CPU 경합이 발생하여 매 테스트마다 표준 편차가 일정하지 않다.

- SLA 99.9%를 달성해야 하는 서비스의 경우, CPU 코어를 격리하여 해당 프로세스에 코어를 매핑해주어야 한다. 

- 코어를 격리한다는 것은 OS의 기본적인 CPU 스케쥴링에서 해당 코어를 제외하는 것을 의미하며 특정 프로세스에 해당 코어를 매핑하지 않는 이상, 격리된 CPU 코어는 절대 사용되지 않는다.

- 정리하자면 여기서는 크게 2가지 설정을 할 예정인데 하나는 CPU 코어를 격리하여 로드밸런싱 프로세스에 해당 코어를 매핑해줄 것이고, 또 다른 하나는 격리된 CPU 코어를 특정 IRQ 전용 코어로 매핑해줄 것이다.

- 특정 IRQ란 간단히 말해서 NIC에 있는 패킷을 메인 메모리로 적재하는 IRQ를 의미하는데 대용량의 트래픽 환경에서는 패킷 DROP을 줄이기 위해 무조건 설정해주어야 한다.

<br/>

<b>1. CPU 코어 격리 (Rocky Linux 8)</b>

- isolate CPU Core

```
vim /etc/default/grub
---------------------------------------------------------------
		...
GRUB_CMDLINE_LINUX="crashkernel=auto rhgb quiet isolcpus=0-12"		// 격리할 CPU 코어 번호 지정
		...
---------------------------------------------------------------
grub2-mkconfig -o /boot/grub2/grub.cfg
systemctl stop irqbalance.service					// irqbalance 데몬은 하드웨어 인터럽트를 여러 CPU 코어에 분산해주는 역할을 하는데
systemctl disable irqbalance.service					// 코어 매핑에 영향을 끼칠 수 있으므로 해당 데몬 중지한다.
sysctl -w "kernel.numa_balancing = 0"
sysctl --system
```

- check isolated CPU Cores

```
cat /sys/devices/system/cpu/isolated
```

<b>2. 로드밸런싱 프로세스에 전용 CPU 코어 매핑</b>

- tasket 커맨드를 이용한 코어 매핑

```
taskset -c 0-3 haproxy -D -f haproxy.cfg
```

- HAProxy Config를 통한 코어 매핑

```
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    daemon
      maxconn 500000
      ulimit-n 1048576
      nbthread 4			// 코어 개수
      cpu-map 1/1-4 0-3			// 코어 매핑
		...
		...
```

<b>3. 패킷 관련 IRQ에 전용 CPU 코어 매핑</b>

- modify Rx, Tx Ring buffer Size and qdisc (backlog, txqueuelen)

```
ethtool -g p2p2 								// 상태 확인
ethtool -G p2p2 rx 2047								// rx 버퍼 사이즈 수정
ethtool -G P2P2 tx 2047								// tx 버퍼 사이즈 수정

sysctl -w "net.core.netdev_max_backlog = 9000000"				// incoming  - backlog Size
ifconfig p2p2 txqueuelen 30000							// Outcoming - txqueuelen Size 
```

- Setting Multi-Rx, Tx queue

```
ethtool -l p2p2									// 상태 확인 (combined이 1이면 Disable 상태, 2 이상이 Multi Queue)
ethtool -L p2p2 combined 4							// Rx,Tx Queue 5개 생성 후 Combined
ethtool -n p2p2 rx-flow-hash tcp4						// NIC에서 Hash 기반으로 패킷을 여러 Rx, Tx queue로 분산한다.
```

- Setting IRQ Affinity

```
cat /proc/interrupt | grep p2p2-TxRx						// 포트 이름에 따라 다를 수도 있으니 처음엔 전체 출력 후 찾아볼 것
echo 6 > /proc/irq/105/smp_affinity_list    					// 105번 IRQ는 격리된 코어 6번으로만 처리한다.                                                 
echo 7 > /proc/irq/106/smp_affinity_list                       			// 106번 IRQ는 격리된 코어 7번으로만 처리한다.
echo 8 > /proc/irq/107/smp_affinity_list					// 107번 IRQ는 격리된 코어 8번으로만 처리한다.
echo 9 > /proc/irq/108/smp_affinity_list					// 108번 IRQ는 격리된 코어 9번으로만 처리한다.
echo 10 > /proc/irq/109/smp_affinity_list					// 109번 IRQ는 격리된 코어 10번으로만 처리한다.
```

- Check IRQ Affinity

```
hping --flood --udp x.x.x.x							// 다른 서버에서 무작위 패킷 전송
watch "cat/proc/interrupt | grep p2p2"						// 전용 코어에서 실행되는지 확인
watch -n 1 "ethtool -S p2p2 | grep -E 'rx_ucast_packets|rx_bcast_packets \	// Hash 기반으로 여러 Multi Queue로 분산되는지 확인
|rx_discards|rx_drops|tx_ucast_packets|tx_bcast_packets|tx_discards|tx_drops'"
```

</details>

<br/>

## 🔀 진행 순서

#### (1) Client VM에서 로드밸런서로 HTTP 부하 테스트

- Ansible 이용하여 Client VM에 Apache Bench 명령 전달

```
ab -c 1000 -n 2000 http://192.168.16.100/dummy_file > bench_log 2>&1
```

#### (2) 각 Client VM에서 진행한 HTTP 부하 테스트 결과 데이터 수집

#### (3) 로드밸런서 서버 상태 체크 후, 추가 테스트 진행

- 부하 테스트에 사용된 Socket이 모두 Closed 되었는 지 확인한다.

```
[root@test_lb ~]# ss -s; sysctl -a | grep nf_conntrack_count
Total: 120220
TCP:   539971 (estab 60999, closed 419947, orphaned 0, timewait 419947)

Transport Total     IP        IPv6
RAW       0         0         0
UDP       12        10        2
TCP       120024    120021    3
INET      120036    120031    5
FRAG      0         0         0

net.netfilter.nf_conntrack_count = 973977
```

<br/>

## 📊 BMT 결과

- Proxy 방식

|Thread|Concurrency|Total Requests|File Size|Requests (sec)|Latency (ms)
|:---:|:---:|:---:|:---:|:---:|:---:|
|1|30,000|90,000|5KiB|13258.52|0.084|
|1|60,000|180,000|5KiB|13587.18|0.074|
|1|90,000|270,000|5KiB|13812.47|0.073|
|1|150,000|450,000|5KiB|12484.59|0.081|
|1|200,000|600,000|5KiB|12520.11|0.081|
|2|30,000|90,000|5KiB|21390.87|0.048|
|2|60,000|180,000|5KiB|20847.13|0.048|
|2|90,000|270,000|5KiB|20561.52|0.049|
|2|150,000|450,000|5KiB|19544.82|0.051|
|2|200,000|600,000|5KiB|19503.01|0.052|
|4|30,000|90,000|5KiB|31800.54|0.032|
|4|60,000|180,000|5KiB|31329.61|0.032|
|4|90,000|270,000|5KiB|32268.92|0.031|
|4|150,000|450,000|5KiB|31361.7|0.032|
|4|200,000|600,000|5KiB|29142.57|0.035|
|8|30,000|90,000|5KiB|37377.91|0.026|
|8|60,000|180,000|5KiB|37297.77|0.029|
|8|90,000|270,000|5KiB|36877.77|0.027|
|8|150,000|450,000|5KiB|36464.39|0.028|
|8|200,000|600,000|5KiB|38769.65|0.026|

<br/>

- L2DSR 방식

|Thread|Concurrency|Total Requests|File Size|Requests (sec)|Latency (ms)
|:---:|:---:|:---:|:---:|:---:|:---:|
|1|30,000|90,000|5KiB|14540.21|0.071|
|1|60,000|180,000|5KiB|14449.97|0.072|
|1|90,000|270,000|5KiB|14667.79|0.072|
|1|150,000|450,000|5KiB|14545.84|0.073|
|1|200,000|600,000|5KiB|14358.51|0.074|

<br/>

## 🌟 트러블 슈팅

<table>
  	<tr>
  		<td align="center">
			문제 상황  
		</td>
		<td>
			HTTP 부하 테스트 시, 로드밸런서에서 다수의 Packet이 DROP 되는 이슈가 발생
		</td>
  	</tr>
	<tr>
		<td align="center">
			원인
		</td>
		<td>
   			NIC 메모리 버퍼에 있는 패킷이 메인 메모리로 적재되는 것 보다 새로 들어오는 패킷이 많아 기존의 패킷이 Overwrite 
		</td>
	</tr>
 	<tr>
		<td align="center">
			해결
		</td>
		<td>
			Packet 관련 IRQ 전용 코어를 매핑 + NIC에서 메모리로 적재하는 패킷의 Bucket Size를 조정하여 해결
		</td>
      	</tr>
</table>

```
[Packet DROP 체크]
[root@test_lb ~]# nstat -az TcpExtListenDrops TcpExtListenOverflows

#kernel
TcpExtListenOverflows           275243             275243.0
TcpExtListenDrops               152465             152465.0

[root@test_lb ~]# ethtool -S ens3
NIC statistics:
     rx_queue_0_packets: 23733047
     rx_queue_0_bytes: 16361520124
     rx_queue_0_drops: 523565
		...
		...
```

<br/>

## 📋 느낀 점

- 5G 미만의 트래픽 환경에서는 기존 프록시 방식에 대비하여 DSR 방식으로 성능 상 10% 내외로 향상 폭이 있었으며, 10G 또는 20G 이상의 대용량 트래픽 환경에서 유의미한 향상 폭이 있을 것으로 추정된다.

- 또한, DSR 방식 로드밸런싱 (ipvsadm)은 기본적으로 커널에서 제공되는 네트워크 모듈을 사용하기 때문에 단일 스레드로 동작하여 멀티 스레드로 동작할 수 있는 Proxy 방식 로드밸런서에 비해 쓰루풋이 낮다는 단점이 있다.

- 따라서 DSR 로드밸런싱을 도입 시, 현재 우리 서비스에서 초당 트래픽이 얼마나 발생하는지 확인한 다음 서버에서 전부 처리하지 못할 대용량의 트래픽이라면 DSR 기능이 제공되는 물리 L4 스위치 장비를 사용해야 한다.

<br/>

## 📄 참고 문헌

- https://docs.vmware.com/en/VMware-NSX-Advanced-Load-Balancer/22.1/Configuration_Guide/GUID-FE309741-DEFF-42C1-9AE1-69F36806E93D.html

- https://www.haproxy.com/blog/haproxy-forwards-over-2-million-http-requests-per-second-on-a-single-aws-arm-instance/  

- https://www.freecodecamp.org/news/how-we-fine-tuned-haproxy-to-achieve-2-000-000-concurrent-ssl-connections-d017e61a4d27/

- https://access.redhat.com/solutions/2144921

- https://discourse.haproxy.org/t/connection-reset-seen-every-2-sec-haproxy/2156/5

- https://techdocs.broadcom.com/us/en/storage-and-ethernet-connectivity/ethernet-nic-controllers/bcm957xxx/adapters/statistics/counters.html

- https://devhicom.tistory.com/4

- https://stackoverflow.com/questions/22491229/load-testing-and-benchmarking-with-siege-vs-wrk









