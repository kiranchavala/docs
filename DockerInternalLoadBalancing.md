#Docker Swarm Internal Load Balancing

This document attempts to explain how load balancing between Docker Swarm services are handled. It is assumed
that you know how to create overlay networks and services in a Docker Swarm. 

Let's look at two services connected to an overlay network. The first service is a web service and the second
service is a REST api. For simplicity, the web service will run as a single instance while the api runs with
two instances.

[image]

Assuming the api service was not specifically created with `--endpoint-mode` to `dnsrr`, the api service should
be reachable from the web service via the hostname which equals the api service name. So, if the api service is
simply named api, the web container will resolve the api hostname to the VIP address of the api service.



