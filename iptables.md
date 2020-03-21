# k8s# 防火墙规则说明
## 规则说明
1. 所有核心业务服务器开放相应的ssh端口(如二部: 业务服务器 22、 数据库10022)
2. lobby、connector服务器只允许游戏外围转发(靠近核心服务器的一层)、主后台外围转发(靠近核心服务器)、同机房业务服务器访问
3. game服务器只允许同机房业务服务器访问(lobby、connector、admin)
4. web-server服务器只允许游戏外围转发(靠近核心服务器的一层)、主后台外围转发(靠近核心服务器)、充值问题转发(靠近核心服务器)、同机房业务服务器访问
5. 所有的数据库服务器只允许核心服务器(所有机房的游戏业务服务器、落地页服务器、产品数据导出服务器)访问
6. 文件服务器只允许游戏核心业务服务器访问
7. 所有游戏转发节点物理第三层(靠近用户的那层)做ipset限制

## 配置规范及操作说明
1. 所有防火墙配置文件 /etc/iptables/iptables.ini
2. 所有数据库服务器防火墙共用一个模版，只要有改动需所有的数据库服务器都需要改
3. 每次有改动都通过操作 /etc/iptables/iptables.ini (初始化之后就不需要使用iptables-save进行保存操作，否则初始配置模版会被覆盖), 改动之后通过 iptables-restore < /etc/iptables/iptables.ini 使防火墙配置生效
4. 如果防火墙涉及到废弃的服务器/VPN信息，需及时清理掉(清理需多次确认)
5. 器配置文件 按照功能份块
* 数据库防火墙配置文件模版(各部门根据实际情况做修改)
```ini
# 允许建立/已经建立的连接及ping
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT

#ssh
-A INPUT -p tcp -m state --state NEW -m tcp --dport 65535 -j ACCEPT

# DBServer
-A INPUT -s 192.168.x.x/32 -j ACCEPT

# Server
-A INPUT -s 192.168.x.x/32 -j ACCEPT

# VPN
-A INPUT -s 192.168.x.x/32 -j ACCEPT

# jumpserver
-A INPUT -s 192.168.x.x/32 -j ACCEPT

# zabbix
-A INPUT -s 192.168.x.x/32 -p tcp -m tcp --dport 10050 -j ACCEPT

# 除上述ip之外，其他的所有ip均拒绝
-A INPUT -j REJECT --reject-with icmp-host-prohibited
COMMIT
```
* 业务服务器防火墙配置文件配置模版(各部门根据实际情况做修改)
```ini
# 允许建立/已经建立的连接及ping
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT

# ssh
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT

# platform1 
-A INPUT -s 192.168.x.x/32 -j ACCEPT

# platform2
-A INPUT -s 192.168.x.x/32 -j ACCEPT

# platform3
-A INPUT -s 192.168.x.x/32 -j ACCEPT

# admin
-A INPUT -s 192.168.x.x/32 -j ACCEPT

# fq
-A INPUT -s 192.168.x.x/32 -j ACCEPT

# vpn
-A INPUT -s 192.168.x.x/32 -j ACCEPT

# jumpserver
-A INPUT -s 192.168.x.x/32 -j ACCEPT

# zabbix
-A INPUT -s 192.168.x.x/32 -p tcp -m tcp --dport 10050 -j ACCEPT

# 除上述ip之外，其他的所有ip均拒绝
-A INPUT -j REJECT --reject-with icmp-host-prohibited
COMMIT
```
*  游戏转发节点防火墙配置文件模版（各部门根据实际情况做修改，只在靠近用户那层做限制）
```bash
connPort=65535
network="eth0"
seconds=1
hitcount=60
speed=1000

apt install ipset -y 
iptables -F
iptables -A INPUT -m set --match-set WHITE_LIST src -j ACCEPT  
iptables -A INPUT -m set --match-set IPSET_LIST src -j DROP
# 限制connPort在 seconds 秒内只能发起hitcount个连接，超过记录丢弃数据包
iptables -A INPUT -m state --state NEW -m recent --name BAD_CC_ACCESS --update -p tcp -m tcp --dport ${connPort} --seconds ${seconds} --hitcount ${hitcount} -j SET --add-set IPSET_LIST src
iptables -A INPUT -m recent --name BAD_CC_ACCESS --set -j ACCEPT 
# 限制网卡network 传出速率为${speed}kb/s
iptables -A OUTPUT -o ${network} -m hashlimit --hashlimit-above ${speed}kb/s --hashlimit-mode dstip --hashlimit-name out -j DROP
```


-A INPUT -s 34.92.160.200/32 -j ACCEPT
-A INPUT -s 34.92.30.163/32 -j ACCEPT
-A INPUT -s 35.220.203.77/32 -j ACCEPT
-A INPUT -s 35.220.138.178/32 -j ACCEPT
-A INPUT -s 35.241.89.38/32 -j ACCEPT
