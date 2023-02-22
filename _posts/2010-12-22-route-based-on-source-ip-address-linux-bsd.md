---
title: "Route based on Source IP Address (Linux / BSD)"
created: 2010-12-22

---

I ran into an interesting situation the other day which I expected would have more documentation online.

Situation: You have a multi-homed router and you would like to route traffic based on client IP addresses, or the source address. In my case I wanted a /24 (Net 1) to be directed (forwarded) through interface A, and another /24 (Net 2) to be forwarded through interface B. In my case I also NATed traffic forwarded to interface A.

This is called source address routing or policy-based routing.

<!--more-->

### Linux / IP

Create two routing tables:

```
ip route flush 11 #interface A
ip route add table 11 to 192.168.0.0/24 dev eth1
ip route add table 11 to default via 192.168.150.1 dev eth1
```

```
ip route flush 10 #interface B
ip route add table 10 to 10.0.0.0/24 dev eth2
ip route add table 10 to default via 10.0.150.1 dev eth2
```

Here we have two interfaces, eth1 and eth2, which are assigned different networks, let's assume that we also have an eth0 which includes clients with both 10.0.0.0/24 and 192.168.0.0/24 addresses.

Then create two rules to send traffic to each table based on the traffic's source IP address:

```
ip rule add from 192.168.0.0/24 table 11 priority 11 #net 1
ip rule add from 10.0.0.0/24 table 10 priority 10 #net 2
```

Now in this example, a default route was assigned to each table. You can add more routing entries, remove the default route, and do whatever. In my scenario both interfaces had a default route and were connected to the Internet. Issuing these commands does not make the routing persistant. You'll need to add this somewhere to your start up scripts.

### OpenBSD / PF

To accomplish this on a BSD system, use PF. You can install PF on FreeBSD, and it ships with OpenBSD. Add the following to the pf.conf file. Then reload pf using pf -F all -f /etc/pf.conf. For more information on pf check the guide.

```
ext\_if="sis0" #interface A
ext\_gw = "192.168.150.1"
int\_if="sis1"
int\_net = "192.168.0.0/24" #net 1
```

With one line we can choose to route from an internal network/interface pair out an external interface/gateway pair.

```
pass in on $int\_if from $int\_net route-to {($ext\_if $ext\_gw)}
```

We can do the same for net 2 and interface B, without using variables.

```
pass in on sis1 from 10.0.0.0/24 route-to {(sis2 10.0.150.1)}
```

These configurations to not NAT route, they just allow a more granular approach to routing/forwarding. Hope this helps.
