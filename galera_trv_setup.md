# Installation GALERA (MYSQL Kluster) Prod TEST (T2)
Först laddas meduim ner för Galera och MYSQL-WSREP

## Konfiguration Diskar

500Gb disk fanns tillgänglig på maskinen(sdb) dessa konfigureras enligt:


```bash
fdisk /dev/sdb
pvcreate /dev/sdb1
vgcreate mysql_vg /dev/sdb1
lvcreate -l 100%FREE -n lv_data mysql_vg
mkdir /data
mkfs.ext4 /dev/mysql_vg/lv_data
mount /dev/mysql_vg/lv_data /data/
```
Sedan läggs denna volym med i /etc/fstab för automount vid uppsart enligt:
```
/dev/mapper/mysql_vg-lv_data /data                   ext4    defaults        1 2
```

## Installation Prerequisits

installera paketen enligt följande:

```bash
yum -y install lsof policycoreutils-python net-tools socat rsync
yum -y install boost-program-options haproxy keepalived
yum -y remove mariadb-libs
rpm -ivh mysql-wsrep-common-5.7-5.7.26-25.18.el7.x86_64.rpm 
rpm -ivh mysql-wsrep-libs-5.7-5.7.26-25.18.el7.x86_64.rpm 
rpm -ivh mysql-wsrep-libs-compat-5.7-5.7.26-25.18.el7.x86_64.rpm
rpm -ivh mysql-wsrep-client-5.7-5.7.26-25.18.el7.x86_64.rpm 
rpm -ivh mysql-wsrep-server-5.7-5.7.26-25.18.el7.x86_64.rpm
rpm -ivh galera-3-25.3.27-2.el7.x86_64.rpm mysql-wsrep-5.7-5.7.26-25.18.el7.x86_64.rpm

```

## Konfigueration
Enable Mysql tjänsten att starta vid Boot
```
systemctl enable mysqld
```
### Firewall (Dock ej på slagen än)
```
firewall-cmd --zone=public --add-service=mysql --permanent
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --zone=public --add-port=4444/tcp --permanent
firewall-cmd --zone=public --add-port=4567/tcp --permanent
firewall-cmd --zone=public --add-port=4567/udp --permanent
firewall-cmd --zone=public --add-port=4568/tcp --permanent
firewall-cmd --reload
```

### SELinux
SELinux är idag Permissive för hela Servern, skulle detta ändras gäller följande konf för Galera samt sätta MySQL demonen till permissive:

```
semanage port -a -t mysqld_port_t -p tcp 4567
semanage port -a -t mysqld_port_t -p udp 4567
semanage port -a -t mysqld_port_t -p tcp 4568
semanage port -a -t mysqld_port_t -p tcp 4444
semanage permissive -a mysqld_t
```

### MySQL
Ändra rättighet på data katalogen (`/data`):
`chown mysql:mysql /data`

Göre en kopia av befintlig konf: `cp /etc/my.cnf /etc/my.cnf.bak`
Ändra sedan `/etc/my.cnf`enligt:

```
[mysqld]
datadir=/data
socket=/var/lib/mysql/mysql.sock
user=mysql
binlog_format=ROW
bind-address=0.0.0.0
default_storage_engine=innodb
innodb_autoinc_lock_mode=2
innodb_flush_log_at_trx_commit=0
innodb_buffer_pool_size=122M
wsrep_provider=/usr/lib64/galera-3/libgalera_smm.so
wsrep_provider_options="gcache.size=300M; gcache.page_size=300M"
wsrep_cluster_name="galera_cluster1"
wsrep_cluster_address="gcomm://192.168.5.179,192.168.5.180"
wsrep_sst_method=rsync
server_id=1
wsrep_node_address="192.168.5.178"
wsrep_node_name="ictdmz189"
[mysql_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

#### Logfil
Skapa logfil genom att:

```
touch /var/log/mysqld.log
chown mysql:mysql /var/log/mysqld.log
```

## Gör föreående steg på alla noder
Förändra endast följande:

```
server_id=1
wsrep_node_address="192.168.5.178"
wsrep_node_name="ictdmz189"
```

samt Ersätt gcomm strängen till att exkludera egenmaskin och lägg på egna

## Uppstart
Starta första noden genom att köra `mysqld_bootstrap`
hitta sedan lösenordet som sattes temporärt i /var/log/messages

Kör sedan `mysql_secure_installation` och följ instruktionerna.

Efter att detta utförst OK starta de öviga noderna genom systemctl start mysqld

### Verifiera
```sql
mysql> SHOW STATUS LIKE 'wsrep_cluster_size';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+
1 row in set (0.00 sec)
```

### Skapa admin konto för remote admin

```sql
mysql –u root –p
CREATE USER 'admin'@'%' IDENTIFIED BY 'K@km0nster';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' WITH GRANT OPTION;
```

## HA och keepalived
### Övervakningskonto i MySQL
Skapa ett konto med korrekt (PROCESS) behörighet i MySQL:
```sql
GRANT PROCESS ON *.* TO 'clustercheckuser'@'localhost' IDENTIFIED BY 'Clust3rCh3ckAassw0rd!';
```

### Clustercheck Script
**/user/bin/clustecheck**
```bash
#!/bin/bash
#
# Based on the original script from Unai Rodriguez
#
if [[ $1 == '-h' || $1 == '--help' ]];then
    echo "Usage: $0 <user> <pass> <available_when_donor=0|1> <log_file> <available_when_readonly=0|1> <defaults_extra_file>"
    exit
fi
# if the disabled file is present, return 503. This allows
# admins to manually remove a node from a cluster easily.
if [ -e "/var/tmp/clustercheck.disabled" ]; then
    # Shell return-code is 1
    echo -en "HTTP/1.1 503 Service Unavailable\r\n"
    echo -en "Content-Type: text/plain\r\n"
    echo -en "Connection: close\r\n"
    echo -en "Content-Length: 51\r\n"
    echo -en "\r\n"
    echo -en "Percona XtraDB Cluster Node is manually disabled.\r\n"
    sleep 0.1
    exit 1
fi
set -e
if [ -f /etc/sysconfig/clustercheck ]; then
        . /etc/sysconfig/clustercheck
fi
MYSQL_USERNAME="${MYSQL_USERNAME:=clustercheckuser}"
MYSQL_PASSWORD="${MYSQL_PASSWORD-Clust3rCh3ckAassw0rd!}"
AVAILABLE_WHEN_DONOR=${AVAILABLE_WHEN_DONOR:-0}
ERR_FILE="${ERR_FILE:-/dev/null}"
AVAILABLE_WHEN_READONLY=${AVAILABLE_WHEN_READONLY:-1}
DEFAULTS_EXTRA_FILE=${DEFAULTS_EXTRA_FILE:-/etc/my.cnf}
#Timeout exists for instances where mysqld may be hung
TIMEOUT=10
EXTRA_ARGS=""
if [[ -n "$MYSQL_USERNAME" ]]; then
    EXTRA_ARGS="$EXTRA_ARGS --user=${MYSQL_USERNAME}"
fi
if [[ -n "$MYSQL_PASSWORD" ]]; then
    EXTRA_ARGS="$EXTRA_ARGS --password=${MYSQL_PASSWORD}"
fi
if [[ -r $DEFAULTS_EXTRA_FILE ]];then
    MYSQL_CMDLINE="mysql -nNE --connect-timeout=$TIMEOUT \
                    ${EXTRA_ARGS}"
else
    MYSQL_CMDLINE="mysql -nNE --connect-timeout=$TIMEOUT ${EXTRA_ARGS}"
fi
#
# Perform the query to check the wsrep_local_state
#
echo $MYSQL_CMDLINE
WSREP_STATUS=$($MYSQL_CMDLINE -e "SHOW STATUS LIKE 'wsrep_local_state';" \
    2>${ERR_FILE} | tail -1 2>>${ERR_FILE})
if [[ "${WSREP_STATUS}" == "4" ]] || [[ "${WSREP_STATUS}" == "2" && ${AVAILABLE_WHEN_DONOR} == 1 ]]
then
    # Check only when set to 0 to avoid latency in response.
    if [[ $AVAILABLE_WHEN_READONLY -eq 0 ]];then
        READ_ONLY=$($MYSQL_CMDLINE -e "SHOW GLOBAL VARIABLES LIKE 'read_only';" \
                    2>${ERR_FILE} | tail -1 2>>${ERR_FILE})
        if [[ "${READ_ONLY}" == "ON" ]];then
            echo -en "HTTP/1.1 503 Service Unavailable\r\n"
            echo -en "Content-Type: text/plain\r\n"
            echo -en "Connection: close\r\n"
            echo -en "Content-Length: 43\r\n"
            echo -en "\r\n"
            echo -en "TRV Galera Cluster Node for MySQL is read-only.\r\n"
            sleep 0.1
            exit 1
        fi
    fi
    # Shell return-code is 0
    echo -en "HTTP/1.1 200 OK\r\n"
    echo -en "Content-Type: text/plain\r\n"
    echo -en "Connection: close\r\n"
    echo -en "Content-Length: 40\r\n"
    echo -en "\r\n"
    echo -en "TRV Galera Cluster Node for MySQL is synced.\r\n"
    sleep 0.1
    exit 0
else
    # Shell return-code is 1
    echo -en "HTTP/1.1 503 Service Unavailable\r\n"
    echo -en "Content-Type: text/plain\r\n"
    echo -en "Connection: close\r\n"
    echo -en "Content-Length: 44\r\n"
    echo -en "\r\n"
    echo -en "TRV Galera Cluster Node for MySQL is not synced.\r\n"
    sleep 0.1
    exit 1
fi
```

### XinitD Service:
**/etc/xinetd.d/mysqlchk**
```
service postgreschk
{
        flags           = REUSE
        socket_type     = stream
        port            = 9201
        wait            = no
        user            = mysql
        server          = /usr/bin/clustercheck
        log_on_failure  += USERID
        disable         = no
        only_from       = 0.0.0.0/0
        per_source      = unlimited
}
```

Där tillhörande Check enligt ovan (Clustercheck)

Här måste vi även lägga till servicen i `/etc/services`
```
echo "mysqlchk	9201/tcp		# ClusterCheck Galero" >> /etc/services
```

### HAProxy
Korrekt config enligt Denna maskin är:
**/etc/haproxy/haproxy.cfg**
```
global
  log /dev/log daemon
  maxconn 32768
  chroot /var/lib/haproxy
  user haproxy
  group haproxy
  daemon
  stats socket /var/lib/haproxy/stats user haproxy group haproxy mode 0640 level operator
  tune.bufsize 32768
  tune.ssl.default-dh-param 2048
  ssl-default-bind-ciphers ALL:!aNULL:!eNULL:!EXPORT:!DES:!3DES:!MD5:!PSK:!RC4:!ADH:!LOW@STRENGTH
 
defaults
  log     global
  option  log-health-checks
  option  log-separate-errors
  option  dontlog-normal
  option  dontlognull
  option  tcplog
  option  socket-stats
  retries 3
  option  redispatch
  maxconn 10000
  timeout connect     5s
  timeout client     50s
  timeout server    450s

frontend DB
  bind 192.168.5.173:3306
  default_backend DB



backend DB
  mode tcp
  balance roundrobin
  option httpchk
  option allbackups
  default-server port 9201 downinter 5s rise 3 fall 2 slowstart 60s maxconn 64 maxqueue 128 weight 100
  server 192.168.5.178 192.168.5.178:3306 check
  server 192.168.5.179 192.168.5.179:3306 check
  server 192.168.5.180 192.168.5.180:3306 check

listen stats 192.168.5.173:8081
     mode http
     option httpclose
     balance roundrobin
     stats uri /
     stats realm Haproxy\ Statistics
     stats auth haproxy:Pr0xy

```

### Keepalived
**/etc/keepalived/keepalived.conf**
```
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}
 
 
vrrp_instance VIP1 {
  interface ens192
  state MASTER
  virtual_router_id 51
  priority 102
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1337
  }
  virtual_ipaddress {
    192.168.5.173
  }
 
  track_script {
    chk_haproxy
  }
}

```

!!Tänk på att sätta korrekt Priority på de olika noderna!!

För att kunna binda till ännu ej assignade adresser måste vi sätta "non local bind" i linux kärnan:

```
echo 1 > /proc/sys/net/ipv4/ip_nonlocal_bind
echo "net.ipv4.ip_nonlocal_bind = 1" > /usr/lib/sysctl.d/05-keepalived-nonlocal-bind.conf
sysctl -p
```



























