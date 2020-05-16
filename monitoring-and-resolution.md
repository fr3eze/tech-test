This tutorial is written in Markdown language, use a Markdown editor to view for best readability. It is available on the link below too:


Digital Ocean's Droplet has been used for demostration purpose on Centos 8

Requirement:
- Create a CentOS VM($10)
- Create 4 block volumes and attach to the VM (2GB each) for RAID 10 purpose

# Monitoring and Resolution

#### Configure Monit 

Install Monit
```
$ dnf install -y git gcc glibc make glibc-devel kernel-headers autoconf automake libtool bison flex libzip-devel pam-devel openssl openssl-devel
$ git clone https://bitbucket.org/tildeslash/monit.git
$ cd monit
$ ./bootstrap
$ ./configure
$ make
$ make install
```

Configure initialisation setup
```
cp monit/monitrc /etc/monitrc
```

Do the following changes in `/etc/monitrc`
```
# Uncomment this line
include /etc/monit.d/*

# Change the localhost to 0.0.0.0 so we can access the web console from anywhere
set httpd port 10080 and
    use address 0.0.0.0  # only accept connection from localhost (drop if you use M/Monit)
    allow  0.0.0.0/0         # allow localhost to connect to the server and
    allow admin:monit      # require user 'admin' with password 'monit'
```

Create a systemd service by creating a file `/lib/systemd/system/monit.service`
```
 [Unit]
 Description=Pro-active monitoring utility for unix systems
 After=network.target
 Documentation=man:monit(1) https://mmonit.com/wiki/Monit/HowTo

 [Service]
 Type=simple
 KillMode=process
 ExecStart=/usr/local/bin/monit -I
 ExecStop=/usr/local/bin/monit quit
 ExecReload=/usr/local/bin/monit reload
 Restart = on-abnormal
 StandardOutput=null

 [Install]
 WantedBy=multi-user.target
```

Start the service
```
$ systemctl daemon-reload
$ systemctl start monit
$ systemctl enable monit
```

Verify if the 10080 port is serving from 0.0.0.0
```
$ ss -tulnp | column -t
...
tcp    LISTEN  0       128     0.0.0.0:10080   0.0.0.0:*     users:(("monit",pid=20524,fd=7))
...
```

You may now access the monit web console via http://<server-ip>s:10080

#### Add monitoring 

Checking blocks can be placed within `/etc/monitrc` as a whole or seperately in `/etc/monit.d/` as we enabled the option earlier. We will use the latter method for better control

For NFS monitoring, create a file `/usr/local/bin/nfsmonit.sh`
```
#!/usr/bin/env bash
df -h | grep -q /raid
```

Create a file `/etc/monit.d/nfsmonit`
```
check program myscript with path "/usr/local/bin/nfsmonit.sh"
   if status != 0 then alert
```

For LUKS monitoring, create a file `/usr/local/bin/luksmonit.sh`
```
#!/usr/bin/env bash
cryptsetup status luks_md0
```

Create a file `/etc/monit.d/luksmonit`
```
check program myscript with path "/usr/local/bin/luksmonit.sh"
   if status != 0 then alert
```

For RAID monitoring, create a file `/etc/monit.d/raidmonit`
```
 check file raid with path /proc/mdstat
   if match "\[.*_.*\]" then alert
```

#### Install Zabbix server
Install Zabbix on a new server following the guide below as I don't see the point creating a whole new document when there is a ready made one.
https://tecadmin.net/install-zabbix-server-centos-8/

#### Create service to monitor load

Create a service writing the current load in /var/log/messages.
Insert block below to a script file `print-load.sh`
```
#!/usr/bin/env bash
/usr/bin/uptime | /usr/bin/logger && uptime
```

Create a file `/etc/systemd/system/print.load.service`
```
[Unit]
Description=Print load to system log
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=root
ExecStart=/usr/bin/env bash /root/print-load.sh

[Install]
WantedBy=multi-user.target
```

Start and enable the new service
```
systemctl start printload ; systemctl enable printload
```

Write a crontab which deletes old logs, cleans /tmp every hour and start the service you just wrote. 
```
$ crontab -l | { cat; echo '0 */1 * * * rm -rf /tmp/*; find /var/log/messages -mtime +7 -exec rm -f {} \; /usr/bin/systemctl start printload'; } | crontab -
```
