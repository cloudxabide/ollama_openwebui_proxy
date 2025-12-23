# Proxy Setup

# Purpose (What)
We cover the setup of the HAProxy node here and review some nuances related to the setup.

![Network Overview](Images/proxy_host_with_vip.png)

Requirements:

* Host/Node with admin privileges
* 2 x IPs (Host IP, Host Virtual IP (VIP))
* OS with haproxy and keepalived


I build my proxy nodes with a Virtual IP (VIP) in addition to, and separate from, the IP of the node itself.  For this I use Keepalived.  There are simpler ways to accomplish, but Keepalived provides additional capability if I ever wanted to make this setup Highly-Available (HA).


Note:  To allow haproxy to bind to the VIP provided and managed by Keepalived, you may need to add a kernel tunable vi sysctl.

```
$ cat /etc/sysctl.d/10-etc_sysctl.d_10-haproxy.conf
net.ipv4.ip_nonlocal_bind = 1
```
