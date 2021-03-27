# otus_OSPF

Таблицы маршрутизации на каждом роутере:

```
me@me-HP-260-G3-DM:~/otus-linux/DZ_OSPF$ vagrant ssh r1
Last login: Sat Mar 27 15:30:00 2021 from 10.0.2.2
[vagrant@r1 ~]$ ip route
default via 10.0.2.2 dev eth0 proto dhcp metric 100 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100 
172.20.10.0/24 dev eth1 proto kernel scope link src 172.20.10.1 metric 101 
172.20.20.0/24 via 192.168.0.2 dev eth2 proto zebra metric 20 
172.20.30.0/24 via 192.168.200.2 dev eth3 proto zebra metric 20 
192.168.0.0/30 dev eth2 proto kernel scope link src 192.168.0.1 metric 102 
192.168.100.0/30 proto zebra metric 20 
	nexthop via 192.168.0.2 dev eth2 weight 1 
	nexthop via 192.168.200.2 dev eth3 weight 1 
192.168.200.0/30 dev eth3 proto kernel scope link src 192.168.200.1 metric 103 
[vagrant@r1 ~]$ 
```

```
me@me-HP-260-G3-DM:~/otus-linux/DZ_OSPF$ vagrant ssh r2
Last login: Sat Mar 27 15:31:09 2021 from 10.0.2.2
[vagrant@r2 ~]$ ip route
default via 10.0.2.2 dev eth0 proto dhcp metric 100 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100 
172.20.10.0/24 via 192.168.0.1 dev eth2 proto zebra metric 20 
172.20.20.0/24 dev eth1 proto kernel scope link src 172.20.20.1 metric 101 
172.20.30.0/24 via 192.168.100.1 dev eth3 proto zebra metric 20 
192.168.0.0/30 dev eth2 proto kernel scope link src 192.168.0.2 metric 102 
192.168.100.0/30 dev eth3 proto kernel scope link src 192.168.100.2 metric 103 
192.168.200.0/30 proto zebra metric 20 
	nexthop via 192.168.0.1 dev eth2 weight 1 
	nexthop via 192.168.100.1 dev eth3 weight 1 
[vagrant@r2 ~]$ 
```

```
me@me-HP-260-G3-DM:~/otus-linux/DZ_OSPF$ vagrant ssh r3
Last login: Sat Mar 27 15:32:19 2021 from 10.0.2.2
[vagrant@r3 ~]$ ip route
default via 10.0.2.2 dev eth0 proto dhcp metric 100 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100 
172.20.10.0/24 via 192.168.200.1 dev eth2 proto zebra metric 20 
172.20.20.0/24 via 192.168.100.2 dev eth3 proto zebra metric 20 
172.20.30.0/24 dev eth1 proto kernel scope link src 172.20.30.1 metric 101 
192.168.0.0/30 proto zebra metric 20 
	nexthop via 192.168.200.1 dev eth2 weight 1 
	nexthop via 192.168.100.2 dev eth3 weight 1 
192.168.100.0/30 dev eth3 proto kernel scope link src 192.168.100.1 metric 103 
192.168.200.0/30 dev eth2 proto kernel scope link src 192.168.200.2 metric 102 
[vagrant@r3 ~]$ 
```

Симметричный роутинг:

```
[vagrant@r1 ~]$ tracepath 172.20.20.1
 1?: [LOCALHOST]                                         pmtu 1500
 1:  172.20.20.1                                           3.730ms reached
 1:  172.20.20.1                                           1.644ms reached
     Resume: pmtu 1500 hops 1 back 1 
[vagrant@r1 ~]$ 
```

Меняем вес на интерфейсе eth2, через который ведёт линк на r2:

```
[vagrant@r1 ~]$ vtysh
vtysh_connect(/var/run/quagga/zebra.vty): stat = Permission denied
[vagrant@r1 ~]$ sudo -i
[root@r1 ~]# vtysh

Hello, this is Quagga (version 0.99.22.4).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

r1# config t
r1(config)# int eth2
r1(config-if)# ip ospf cost 1000
r1(config-if)# exit
r1(config)# exit
r1# exit
[root@r1 ~]# exit
logout
[vagrant@r1 ~]$ tracepath 172.20.20.1
 1?: [LOCALHOST]                                         pmtu 1500
 1:  192.168.200.2                                         5.541ms 
 1:  192.168.200.2                                         1.864ms 
 2:  172.20.20.1                                           3.389ms reached
     Resume: pmtu 1500 hops 2 back 2 
[vagrant@r1 ~]$ 
```

Симметричность можно вернуть, изменив вес на r3 для интерфейса eth3:

```
[vagrant@r3 ~]$ sudo -i
[root@r3 ~]# vtysh

Hello, this is Quagga (version 0.99.22.4).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

r3# config t
r3(config)# int eth3
r3(config-if)# ip ospf cost 1000
r3(config-if)# exit
r3(config)# exit
r3# exit
[root@r3 ~]# 
```

Тогда:

```
[vagrant@r1 ~]$ tracepath 172.20.20.1
 1?: [LOCALHOST]                                         pmtu 1500
 1:  172.20.20.1                                           1.702ms reached
 1:  172.20.20.1                                           1.872ms reached
     Resume: pmtu 1500 hops 1 back 1 
[vagrant@r1 ~]$ 
```

Примечание: почему-то не сработала опция в .yml-файлах Ansible:

    - name: edit sysctl rp_filter
      sysctl:
        name: net.ipv4.conf.all.rp_filter
        value: '0'
        sysctl_set: yes
        state: present
        reload: yes
        
Пришлось выставлять отдельно для каждого интерфейса:

    - name: edit sysctl rp_filter
      sysctl:
        name: net.ipv4.conf.eth2.rp_filter
        value: '0'
        sysctl_set: yes
        state: present
        reload: yes

