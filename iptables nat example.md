#### 网络拓扑

```
                     ____________
                    |            |
                    |  client    |
                    |____________|                     
                    nat=172.16.214.129 (eth0)             
                          |                           
                          |                           
                    nat=172.16.214.130 (eth0)             
                    ____________                      
                    |            |                     
                    | nat-server |                     
                    |____________|                     
                    vnet1=192.168.100.1 (eth1)         
                          |                           
                       (switch)
                          |                        
                    vnet1=192.168.100.20 (eth0)      
                    _____________              
                    |             |             
                    | realserver1 |             
                    |_____________|            
```

- nat-server：` sysctl  -w net.ipv4.ip_forward=1`
- realserver：`ip route add default via 192.168.100.1`
- client：`ip route add 192.168.100.0/24 via 172.16.214.130 dev eth0`

```
~]#  curl 192.168.100.20
<h1> Backend Server </h1>
```

#### 添加 FORWARD 规则

```
~]# iptables -P  FORWARD   DROP
```

```
~]# iptables -A FORWARD -d 192.168.100.20 -p tcp --dport 80 -j  ACCEPT
~]# iptables -A FORWARD -s 192.168.100.20 -p tcp --sport 80 -j  ACCEPT
```

```
~]# iptables -A FORWARD -m state --state ESTABLISHED -j ACCEPT
~]# iptables -A FORWARD -d 192.168.100.20 -p tcp -m multiport  --dports 22,80 -m state --state NEW -j  ACCEPT
```

```
~]# modprobe nf_conntrack_ftp
~]# iptables -R FORWARD 1 -m state --state ESTABLISHED,RELATED -j ACCEPT
~]# iptables -R FORWARD 2 -d 192.168.100.20 -p tcp -m multiport --dports 21,22,80  -m state --state NEW -j ACCEPT
~]# iptables -vnL // 开放内网的ftp给外网
Chain INPUT (policy ACCEPT 69 packets, 7308 bytes)
pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy DROP 0 packets, 0 bytes)
pkts bytes target     prot opt in     out     source               destination         
  39  2493 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
   1    60 ACCEPT     tcp  --  *      *       0.0.0.0/0            192.168.100.20       multiport dports 21,22,80 state NEW

Chain OUTPUT (policy ACCEPT 49 packets, 7260 bytes)
pkts bytes target     prot opt in     out     source               destination     
```

#### Network Address Translation(网络地址转换)

```
// SNAT: 只修改请求报文的源地址，发生在 POSTROUTING（内网访问外网）,我们删除路由规则后，添加 snat 规则，还是内网还是可以正常访问外网
ip route del  192.168.100.0/24 via 172.16.214.130 dev eth0
~]# iptables -t nat -A POSTROUTING -s 192.168.100.0/24  ! -d 192.168.100.0/24  -j SNAT --to-source 172.16.214.130
realserver ~]# curl  172.16.214.129 # 验证

// DNAT：只修改请求报文的目标地址，发生在 PREROUTING（外网访问内网）
 ~]# iptables -t nat -A PREROUTING -d 172.16.214.130 -p tcp --dport 80 -j DNAT  --to-destination 192.168.100.20
 client ~]# curl 172.16.214.130

// 假设内网 ssh 端口为 22122，那么我们可以通过一下规则实现 ssh 转发
~]# iptables -t nat -A PREROUTING -d 172.16.214.130 -p tcp --dport 22122  -j DNAT  --to-destination 192.168.100.20:22122
➜  ~ ssh root@172.16.214.130 -p 22122

// MASQUERADE：地址伪装(内网访问外网)，当前系统用的是ADSL/3G/4G动态拨号方式，那么每次拨号，出口IP都会改变，SNAT就会有局限性，
~]# iptables -t nat -A POSTROUTING -s 192.168.100.0/24  ! -d 192.168.100.0/24  -j MASQUERADE
realserver ~]# curl  172.16.214.129
```
