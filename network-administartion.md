This tutorial is written in Markdown language, use a Markdown editor to view for best readability.

Digital Ocean's Droplet has been used for demostration purpose on Centos 8

Requirement:
- Create a CentOS VM($10)
- Create 4 block volumes and attach to the VM (2GB each) for RAID 10 purpose

# Network Administration

#### Configure Bond interface

Check available network interfaces to create bond
```
$ nmcli device show
```

Create bond connection 
```
nmcli connection add type team con-name bond0 ifname bond0 config '{"runner": {"name": "activebackup"}}'
```

Configure the bond interface
```
$ nmcli con mod bond0  ipv4.addresses 10.1.12.4/16 ; \
  nmcli con mod bond0  ipv4.gateway 10.1.0.1 ; \
  nmcli con mod bond0  ipv4.dns "8.8.8.8 8.8.4.4" ; \
  nmcli con mod bond0  ipv4.method manual ; \
  nmcli con mod bond0  connection.autoconnect yes ; \
```

Attach interfaces to the bond interface as slave in a active-backup relationship
```
$ nmcli con add type team-slave con-name bond0-slave0 ifname ens3 master bond0
$ nmcli con add type team-slave con-name bond0-slave1 ifname ens4 master bond0
```
Restart the bond interface to take effect
```
$ nmcli connection down bond0 && nmcli connection up bond0
```

A few ways to verify the bond interface is successfully set up
```
$ ip a
$ teamdctl bond0 state
```

#### Configure VPN

Just refer to the ready-made guide below and substitute the required server and clients IP, don't feel like reinventing the wheel copying another copy of my version as this would be suffice for production needs:
 https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-an-openvpn-server-on-centos-8 

#### Configure IPtables

It is important to implement from source 10.12.65.128 if you are in a SSH connection, else the session will be terminated instantly
```
$ iptables -A INPUT -p tcp -s 10.12.65.128 --match multiport --dport 22,80,443 -j ACCEPT
```

Set default INPUT policy to DROP
```
iptables -P INPUT DROP
```

Verify curren IPtables rules
```
$ iptables -L
Chain INPUT (policy DROP)
target     prot opt source               destination
ACCEPT     tcp  --  10.12.65.128         anywhere             multiport dports ssh,http,https

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```
