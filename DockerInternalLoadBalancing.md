# Docker Swarm Internal Load Balancing

This document attempts to explain how load balancing between Docker Swarm services are handled. It is assumed
that you know how to create overlay networks and services in a Docker Swarm. 

Let's look at two services connected to an overlay network. The first service is a web service and the second
service is a REST api. For simplicity, we assume that the web service runs with a single instance while the api 
runs with two instances.

[image]

Assuming the api service was not specifically created with `--endpoint-mode` to `dnsrr`, the api service will be
reachable from the web service on the hostname which is the very same as the api service name. So, if api service 
name is `api` the web container may resolve the hostname `api` to the VIP address of the api service.

## Who is on the other end?

The VIP address which the `api` hostname resolves leads to somewhere. After all, we know that load balancing takes
place and that traffic to the api service get distributed onto the two instances of the api service. 

Let's look at iptables in the web container. Let's use nsenter to switch to the namespace of the web container.

`sudo nsenter --net=/run/docker/netns/83c5b5bffa3f`

Now that we've switched, let's look at iptables.

```
root@docker02:~# iptables -vL -t mangle
Chain PREROUTING (policy ACCEPT 313K packets, 29M bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 313K packets, 29M bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 558K packets, 45M bytes)
 pkts bytes target     prot opt in     out     source               destination         
 852K   57M MARK       all  --  any    any     anywhere             172.50.0.20          MARK set 0x128
   12   788 MARK       all  --  any    any     anywhere             172.50.0.18          MARK set 0x13b
 231K   19M MARK       all  --  any    any     anywhere             172.50.0.37          MARK set 0x13d

Chain POSTROUTING (policy ACCEPT 326K packets, 26M bytes)
 pkts bytes target     prot opt in     out     source               destination         
```
