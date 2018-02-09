# Docker Swarm Internal Load Balancing

This document attempts to explain how load balancing between Docker Swarm services are handled. It is assumed
that you know how overlay networks and services work in a Docker Swarm. 

Let's look at two services connected to an overlay network. The first service is a web service and the second
service is a REST api. For simplicity, we assume that the web service runs as a single instance while the api 
runs as two instances.

Assuming the api service was not specifically created with `--endpoint-mode` to `dnsrr`, the api service will be
reachable from web task(s) via the api service name and aliases. If the api service name is `api` the web task 
may resolve the hostname `api` to the VIP address of the api service.

Let's validate this from within the web task.

```
root@e7905051b7bb:/# dig api | grep "\sA\s"
api.			600	IN	A	172.50.0.37
```

Looking at the above output, we see that `172.50.0.37` is te VIP address for the api service.


## Who is on the other end?

The VIP address `172.50.0.37` which the `api` hostname resolves to leads to somewhere. After all, we know that load 
balancing takes place and that traffic to the `api` service get distributed onto the two `api` tasks.

Let's look at iptables in the web container. Let's use nsenter to switch to the namespace of the web task.

`administrator@docker02:~$ sudo nsenter --net=/run/docker/netns/83c5b5bffa3f`

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

So far so good, we know that the web task resolves the hostname `api` to `172.50.0.37`. We also know that packets
with destination `172.50.0.37` get marked with `0x13d`. 

## Who's looking for that mark?

Packets going to `172.50.0.37` get marked with `0x13d`. Let's head over to `ipvsadm` to see how that mark is being used.

We're still in the web task's network namespace.

```
root@docker02:~# ipvsadm -L -n --stats
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
  -> RemoteAddress:Port
FWM  296                            105116   827026   774414 55260589 51457957
  -> 172.50.0.21:0                   92972   729968   711766 49472933 47539945
FWM  315                                 2       12        8      788      830
  -> 172.50.0.34:0                       2       12        8      788      830
FWM  317                             44981   231280   218415 19264401 22763535
  -> 172.50.0.40:0                   21936   109838   109631  9246909 11278325
  -> 172.50.0.41:0                   21937   115545   103936  9544368 10970396
```

What a good match! Packets marked with `0x13d` (or 317) gets load balanced to `172.50.0.40` or `172.50.0.41`. The load
balancing is taken care of inside the web task by IPVS. 

## That's it!

That is all there is to it. Docker's DNS will resolve service names to a VIP address. Packets going to that VIP will be
marked by iptables, and subsequently picked up by IPVS. IPVS will then take care of the actual load balancing. This all
happens within the task where packets originate from.
