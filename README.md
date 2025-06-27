# Install Openshift with Agent Based Installer
### [I. Overview](#overview)
- [1. Architecture](#architect)
- [2. IP Planning](#ipplanning)
- [3. Topology](#topo)

### [II. Installing](#enviroment)
- [1. Install tool client on Bastion node](#bastion)
- [2. Install Registry](#registry)
- - [2.1. Registry use Harbor](#harbor)
- - [2.2. Registry use Red Hat Quay](#quay)
- [3. Disconnected Registry Mirroring ](#mirror)
- [4. Installing Cluster Based on this Disconnected Registry](#install)
- - [4.1. Create install-config.yaml](#install-config)
- - [4.2. Create agent-config.yaml](#agent-config)
- [5. Create the ISOs needed for install](#iso)



## Set timezone 
### Ulimit
```
ULIMITS_CONF_PATH="/etc/security/limits.conf"

sed -i -r "/^\*(.*)soft(.*)nofile(.*)/d" $ULIMITS_CONF_PATH
sed -i -r "/^\*(.*)hard(.*)nofile(.*)/d" $ULIMITS_CONF_PATH
sed -i -r "/^\*(.*)soft(.*)nproc(.*)/d" $ULIMITS_CONF_PATH
sed -i -r "/^\*(.*)hard(.*)nproc(.*)/d" $ULIMITS_CONF_PATH
sed -i -r "/^\*(.*)soft(.*)memlock(.*)/d" $ULIMITS_CONF_PATH
sed -i -r "/^\*(.*)hard(.*)memlock(.*)/d" $ULIMITS_CONF_PATH

echo "* soft nofile 655360" >> $ULIMITS_CONF_PATH
echo "* hard nofile 131072" >> $ULIMITS_CONF_PATH
echo "* soft nproc 655360" >> $ULIMITS_CONF_PATH
echo "* hard nproc 655360" >> $ULIMITS_CONF_PATH
echo "* soft memlock unlimited" >> $ULIMITS_CONF_PATH
echo "* hard memlock unlimited" >> $ULIMITS_CONF_PATH


timedatectl set-timezone "Asia/Ho_Chi_Minh"
timedatectl set-ntp 1
```

Tunning Sysctl
```
# Network setup
###################

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
# SWAP settings
vm.swappiness=0
vm.panic_on_oom=0
vm.overcommit_memory=1
kernel.panic=10
kernel.panic_on_oops=1
vm.max_map_count = 262144

# Have a larger connection range available
net.ipv4.ip_local_port_range=1024 65000

# Increase max connection
net.core.somaxconn=10000

# Reuse closed sockets faster
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_fin_timeout=15

# The maximum number of "backlogged sockets".  Default is 128.
net.core.somaxconn=4096
net.core.netdev_max_backlog=4096

# 16MB per socket - which sounds like a lot,
net.core.rmem_max=16777216
net.core.wmem_max=16777216

# Various network tunables
net.ipv4.tcp_max_syn_backlog=20480
net.ipv4.tcp_max_tw_buckets=400000
net.ipv4.tcp_no_metrics_save=1
net.ipv4.tcp_rmem=4096 87380 16777216
net.ipv4.tcp_syn_retries=2
net.ipv4.tcp_synack_retries=2
net.ipv4.tcp_wmem=4096 65536 16777216

# ARP cache settings for a highly loaded docker swarm
net.ipv4.neigh.default.gc_thresh1=8096
net.ipv4.neigh.default.gc_thresh2=12288
net.ipv4.neigh.default.gc_thresh3=16384

# ip_forward and tcp keepalive for iptables
net.ipv4.tcp_keepalive_time=600
net.ipv4.ip_forward=1

# monitor file system events
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
```
### Habor Registry
Download package from link Github Harbor https://github.com/goharbor/harbor/releases

harbor.yml
```
hostname: registry.hub.bca.gov.vn
http:
  port: 80
https:
  port: 443
  certificate: /opt/harbor/cert/cert.pem
  private_key: /opt/harbor/cert/key.pem
harbor_admin_password: Harbor12345
database:
  password: root123
  max_idle_conns: 100
  max_open_conns: 900
  conn_max_lifetime: 5m
  conn_max_idle_time: 0
data_volume: /data
trivy:
  ignore_unfixed: false
  skip_update: false
  skip_java_db_update: false
  offline_scan: false
  security_check: vuln
  insecure: false
  timeout: 5m0s
jobservice:
  max_job_workers: 10
  max_job_duration_hours: 24
  job_loggers:
    - STD_OUTPUT
    - FILE
  logger_sweeper_duration: 1
notification:
  webhook_job_max_retry: 3
  webhook_job_http_client_timeout: 3
log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /var/log/harbor
_version: 2.13.0
proxy:
  http_proxy:
  https_proxy:
  no_proxy:
  components:
    - core
    - jobservice
    - trivy
upload_purging:
  enabled: true
  age: 168h
  interval: 24h
  dryrun: false
cache:
  enabled: false
  expire_hours: 24
```
### Install Red Hat Quay
Download Red Hat Quay
```
wget https://mirror.openshift.com/pub/cgw/mirror-registry/latest/mirror-registry-amd64.tar.gz
tar -xzvf mirror-registry-amd64.tar.gz
```
Chạy lệnh sau để instal Red Hat Quay
```
export REGISTRY_SERVER = "registry.<domain>.com"
./mirror-registry install --quayHostname $REGISTRY_SERVER --quayRoot /data/quay --quayStorage /data/quay/storage --sqliteStorage /data/quay/sqlitedb --initUser huynd --initPassword xxxxxxxxx
```

Sau khi Red Hat quay thực hiện cài đặt xong thì cần thực hiện TrustCA để truy cập đến Quay bằng HTTPS
```
cp /data/quay/quay-rootCA/rootCA.pem /etc/pki/ca-trust/source/anchors/

update-ca-trust extract
```

### Download Openshift client tool
```
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.19.0/openshift-client-linux-4.19.0.tar.gz
tar -xvf openshift-client-linux-4.19.0.tar.gz

wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest/oc-mirror.rhel9.tar.gz
tar -xzvf oc-mirror.rhel9.tar.gz

wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.19.0/openshift-install-linux-4.19.0.tar.gz
tar -xvf openshift-install-linux-4.19.0.tar.gz
```
### Create pull-secret
Vào link https://console.redhat.com/openshift/downloads#tool-pull-secret để lấy pull-secret
```
yum install jq -y
cat pull-secret.txt | jq > pull-secret.json
mkdir /root/.docker/
cp pull-secret.json /root/.docker/auth.json
```

### Create directory openshift

```
mkdir -p openshift/install
```
Tạo 2 file agent-config.yaml và install-config.yaml
```
cd openshift/install
```
Nội dung file install-config.yaml
```
apiVersion: v1
baseDomain: ocp-poc-demo.com
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: disconnected
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.4.0/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  baremetal:
    apiVIP: "192.168.4.60"
    ingressVIP: "192.168.4.61"
pullSecret: '{"auths":{"host2.ocp-poc-demo.com:8443":{"auth":"aW5pdDpPztrFNTRoMEM3b0g5WnUxR3Y4RtNJVldVTnpENlh5QQ=="}}}'
sshKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDY4FFXOr3y52gyDVGeZvtf+Le3NLkllsGqJfKLWQOpHpL7AFlibK/YBlZMD5yT//ZVJ1g+AreSZpcbngvcVFdMhSL8eFSHGDAT1aZIRVdPnQ/3eNrLsLjldiR+HT191QEHl3DHY0ZyT37rl30CQ5xiUd7/5ZemiArypnxjelsaYIEnT5UcRDOAk3CDPbF8M0gNNwNAqKmRGNKcJmTgSSLjJGDblxNGMk5TIl21yxNZqJqDdQgj4s9cp0OM5ZPg1ZE9syKSqg/PkNcOZ5PVzFrOpVsVJTiSnTPlU5bmqbrwnH3ij2QdtSBh4fznchaY6z38uT8EBzXycuvaILLVVl2Qa/mITjujMVicnDYpilUeoDnd6/ZR6QZfKnCBz/T4MQ24wC2TlxXqtNSpaIbAOsNovbOvDmt5iDD1gsRGXfyrLjDAzUJKHOKWGkLbcolGjSemqzvfeDU/7iA90zUhGORmbawQcQeRHEwzIXzpL+Sby3ntrx4a/KI5STw9rCTAsPE= keith@host3.ocp-poc-demo.com'
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  MIIDcTCCAlmgAwIBAgIUDOyG3KLLwQsVkSahzbuaHFEZxU4wDQYJKoZIhvcNAQEL
  BQAweTELMAkGA1UEBhMCVVMxCzAJBgNVBAgMAlZBMREwDwYDVQQHDAhOZXcgWW9y
  azENMAsGA1UECgwEUXVheTERMA8GA1UECwwIRGl2aXNpb24xKDAmBgNVBAMMH3Rl
  c3R2bS1kbnZyLnZvaXAuY2hhcnRlcmxhYi5jb20wHhcNMjUwNTE2MjAyNzEyWhcN
  MjYwNTA3MjAyNzEyWjAaMRgwFgYDVQQDDA9xdWF5LWVudGVycHJpc2UwggEiMA0G
  CSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDN0HPGI7rR6DuvcfFzOzA45AHjoKDY
  kfiSCRzVRtm8SwA3ckORMkaGtTcPne9xPBoWZGBSBwRIzc2sLuwaMVs7cqavSHeo
  x8jYdUG1esnSdyfAOvLN/+gjvH9f6b6S6fCE+RWoI0YoMg8CNV3kOLOh46XDHWuM
  eVSz2n2b5Ni/zufZO6S9ht/QR0VBn4J0DSnrRtc4jvi/1AM8+YFRF7e1n0jqTGyv
  u/cXn9kisbvH9ouZ/0weX7uz4E00VLRiX4a5RZlpNzSlwb6BRINh2HwE1bCUx35p
  QWhZpiE7A9xMlHs6jfXHTMIRW9+AgxXL6PzBubqWIngk5U7+8yU+n5avAgMBAAGj
  UDBOMAsGA1UdDwQEAwIC5DATBgNVHSUEDDAKBggrBgEFBQcDATAqBgNVHREEIzAh
  gh90ZXN0dm0tZG52ci52b2lwLmNoYXJ0ZXJsYWIuY29tMA0GCSqGSIb3DQEBCwUA
  A4IBAQAT/WKxvtn59MnaO7nz+kfWrekPmr8W0vJiC04b3goWfnhdjGW/NXCuutio
  K8ESyazUfTaWYLHkbq/Yx+A3I/7T5aA9rFPkXhIE4KRWGmFciMXPfmKvCBhsLhrx
  toL7WD5dthrLJbeRlzaG0zXwf2IAa3pzx72SBdBh81cW6UY88O6DwlIVN66tMpIk
  P2FWUoZ22pHCKMkNSh8y1xfL3ddP0OhlcsobS000E4vgXxeGrFEKkyuYpP8TsPzZ
  SdlJwXWIl3vhpmLdFiNXd5Td81/UYbh62t/A9ZzS1lY55cGY7C6BvN32RUjkM2Z/
  BCmhQXTqg5ar6O/nct2VXKIuvtMl
  -----END CERTIFICATE-----
  -----BEGIN CERTIFICATE-----
  MIID5DCCAsygAwIBAgIUMy2WGoXqDOQbrKb8it7Vyi7D0iMwDQYJKoZIhvcNAQEL
  BQAweTELMAkGA1UEBhMCVVMxCzAJBgNVBAgMAlZBMREwDwYDVQQHDAhOZXcgWW9y
  azENMAsGA1UECgwEUXVheTERMA8GA1UECwwIRGl2aXNpb24xKDAmBgNVBAMMH3Rl
  c3R2bS1kbnZyLnZvaXAuY2hhcnRlcmxhYi5jb20wHhcNMjUwNTE2MjAyNzEwWhcN
  MjgwMzA1MjAyNzEwWjB5MQswCQYDVQQGEwJVUzELMAkGA1UECAwCVkExETAPBgNV
  BAcMCE5ldyBZb3JrMQ0wCwYDVQQKDARRdWF5MREwDwYDVQQLDAhEaXZpc2lvbjEo
  MCYGA1UEAwwfdGVzdHZtLWRudnIudm9pcC5jaGFydGVybGFiLmNvbTCCASIwDQYJ
  KoZIhvcNAQEBBQADggEPADCCAQoCggEBAKSWa+ySDNORb7UprChr1nT8WxcEdyK6
  gkvwZ2TBhU/rqDqZLMqyIvtFn57xgHpOhseyCnOVAN7BRJDx09xZR9o/O9oqFu2B
  sZgcTEI8Rc3/gH8AnTaXVISr1XXOwDiS7ILRlaT+TsCVim59qJ1H1KNEk5kDROJr
  Ol/1wx7cnNcKFQgwKzf6LgfLYcAdOxLMvl9CBVxSATpvyf2C6CnimhaNYDbpXmKp
  PLaWIvRwpnd4ZsjpVbECH6s9K8MAQ47YGvkbnVlDOYStUUA4qhoXuxueFZk/Sp+K
  ugR0uNtdaf79u0Kd0yInTMrAOQPpUKNJ2LX/fdGH6xA3TDnNpQgcL/sCAwEAAaNk
  MGIwCwYDVR0PBAQDAgLkMBMGA1UdJQQMMAoGCCsGAQUFBwMBMCoGA1UdEQQjMCGC
  H3Rlc3R2bS1kbnZyLnZvaXAuY2hhcnRlcmxhYi5jb20wEgYDVR0TAQH/BAgwBgEB
  /wIBATANBgkqhkiG9w0BAQsFAAOCAQEAKkUirlO+agjwL1c4DOv/ulYu6CdQQ4tM
  +sK2cftREQsLG+85++8/hXOmUmuWmR+NfOVLSumwJXw8Hn4/HK7oBU3uQdIOsEYl
  tcUH9tcGXfHvnnGKSTmCTVpZ99wvWMfvpLvnBu5G6x/bXQ3KUza7GdjpVorI3RHW
  K9hVL/buFgplU5reJRXndxXRw0+Q2O5dNn0UsClUklrabDlEBPjiXKYeoiTdRfMl
  x7pU+hO0L9JYdcN6XfGSB/CbXhugXajPaYAf8wcqSCEZWzsL14cBznKRDbYfk2sZ
  8m2IwgGdLbR9dBjxaquNVWm/0daqMIG4+paN0RLJKhY0jxt88BKKqQ==
  -----END CERTIFICATE-----
imageDigestSources:
- mirrors:
  - host2.ocp-poc-demo.com:8443/openshift/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - host2.ocp-poc-demo.com:8443/openshift/release-images
  source: quay.io/openshift-release-dev/ocp-release
```
Nội dung file agent-config.yaml
```
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: cluster-agent-config
rendezvousIP: 192.168.4.200
hosts:

  - hostname: host1
    interfaces:
      - name: eth0
        macAddress: 52:54:00:bc:e6:73
    networkConfig:
      interfaces:
        - name: eth0
          type: ethernet
          state: up
          mtu: 1500
          ipv4:
            enabled: true
            dhcp: false
            address:
              - ip: 192.168.4.200
                prefix-length: 24
          ipv6:
            enabled: false
            dhcp: false
            autoconf: false
      dns-resolver:
        config:
          search:
            - disconnected.ocp-poc-demo.com
          server:
            - 8.8.8.8
            - 8.8.4.4
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.4.1
            next-hop-interface: eth0
            table-id: 254

  - hostname: host3
    interfaces:
      - name: eth0
        macAddress: 52:54:00:6d:62:9b
    networkConfig:
      interfaces:
        - name: eth0
          type: ethernet
          state: up
          mtu: 1500
          ipv4:
            enabled: true
            dhcp: false
            address:
              - ip: 192.168.4.202
                prefix-length: 24
          ipv6:
            enabled: false
            dhcp: false
            autoconf: false
      dns-resolver:
        config:
          search:
            - disconnected.ocp-poc-demo.com
          server:
            - 8.8.8.8
            - 8.8.4.4
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.4.1
            next-hop-interface: eth0
            table-id: 254
            
  - hostname: host6
    interfaces:
      - name: eth0
        macAddress: 52:54:00:18:10:5b
    networkConfig:
      interfaces:
        - name: eth0
          type: ethernet
          state: up
          mtu: 1500
          ipv4:
            enabled: true
            dhcp: false
            address:
              - ip: 192.168.4.206
                prefix-length: 24
          ipv6:
            enabled: false
            dhcp: false
            autoconf: false
      dns-resolver:
        config:
          search:
            - disconnected.ocp-poc-demo.com
          server:
            - 8.8.8.8
            - 8.8.4.4
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.4.1
            next-hop-interface: eth0
            table-id: 254
```

Chạy câu lệnh để create file iso boot VM
```
openshift-install agent create image --dir ./install
```
Sau khi boot VM sử dụng lệnh dưới để theo dõi tình trạng init cluster
```
openshift-install agent --dir install wait-for bootstrap-complete --log-level=debug

openshift-install agent --dir install wait-for install-complete --log-level=debug
```
