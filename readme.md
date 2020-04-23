# A Guide to setup a HA Proxy and Keepalived for a HA load balancer
## Overview
This guide show you how to seup a HA load balancer using HAproxy and Keepalived
## Setup
Create 4 Ubuntu VMs:
| Server Name   |  IP Address 
|---|---|---|---|---|
|LB1   |  192.168.127.150 
|LB2   |  192.168.127.151 
|Webserver1  |  192.168.127.152 
|Webserver2  |   192.168.127.153
We will use a Virtual IP address:192.168.127.154

### Configure webserver
On the webservers install apache2 
```
sudo apt install apache2
```
edit the default HTML page on webserver1
```
 sudo vi /var/www/html/index.html
```
Modify the following line:
``` 
background-color: #FF0000;
```
On Webserver2 modify the same file and modfiy to:
``` 
background-color: #00FFFF;
```
You should be able to visit http://192.168.127.152 and http://192.168.127.153, Webserver1 should bring back the red web page and webserver2 should bring back the blue website.
### Configure Load Balancer
On LB1 install HA Proxy and Keepalived
```
sudo apt install haproxy
apt-get install keepalived
```
Edit the haproxy config:
```
 sudo vi /etc/haproxy/haproxy.cfg
```
Add the following
```
global
        log 127.0.0.1   local0
        log 127.0.0.1   local1 notice
        #log loghost    local0 info
        maxconn 4096
        #debug
        #quiet
        user haproxy
        group haproxy

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        retries 3
        redispatch
        maxconn 2000
        contimeout      5000
        clitimeout      50000
        srvtimeout      50000

frontend www
       bind 192.168.127.154:80
       default_backend al_pool

backend al_pool
       balance roundrobin
       server webA 192.168.127.152:80 check
       server webB 192.168.127.153:80 check
```
Edit the keepalived config
```
vi /etc/keepalived/keepalived.conf
```
Add the following
```
vrrp_script chk_haproxy {           # Requires keepalived-1.1.13
        script "killall -0 haproxy"     # cheaper than pidof
        interval 2                      # check every 2 seconds
        weight 2                        # add 2 points of prio if OK
}

vrrp_instance VI_1 {
        interface eth0
        state MASTER
        virtual_router_id 51
        priority 101                    # 101 on master, 100 on backup
        virtual_ipaddress {
            192.168.127.154
        }
        track_script {
            chk_haproxy
        }
}
```
Restart the HAproxy and keepalivd
```
sudo service haproxy restart
sudo service keepalived restart
```
If you vists http://192.168.127.154 you should get one of the websites the blue or red one. if you should shut down the server you visted. HA proxy should failover the HTTP connction to the other web server.

On the second load balancer install HA Proxy and Keepalived follow the above steps expect when you edit the /etc/keepalived/keepalived.conf file, change the fellowing line:
```
   priority 100                    # 101 on master, 100 on backup
```

Now when you stop the HAproxy on load balancer1, load balancer 2 should start HAproxy server and take the ownership of 192.168.127.154 IPaddress
