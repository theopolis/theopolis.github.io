---
title: "Gelf: L1 Emulation, L2 Tunneling, using an HTTP Client"
created: 2012-04-10
tags: 
  - iphone
  - networking
  - tether
  - fun

image: /assets/images/img.png
---

Simply: Gelf uses an HTTP client to bridge two or more networks. The iPhone is the primary use case; it has access to both AT&T's mobile network as well as an ad-hoc network. You can bridge the two using Gelf, without running any code on the iPhone, aside from client-side HTML and JavaScript.

This achieves a non-jailbroken, non-rooted, poor-man's network tether. Here's the catch, Gelf needs to run on a device inside each target network. Gelf functions as the L2 tunnel end-points, and the L1 emulation: achieved through an HTTP client.

<!--more-->

### Example (using EC2)

1. Run Gelf on an EC2 host (**server1**)
2. Run Gelf on your Internet-handicapped laptop (**server2**)
3. Create an ad-hoc network from your laptop to an iPhone
4. Browse to the Gelf webserver (**server2**) from the iPhone
5. Use the web interface and enter the IP/port of **server1**

Result: you can route traffic between **server1**/**server2**. Simple? I hope so.

### **How easy?

Requirements: _Cherrypy >= 3.2.2_, PyCrypto (for optional encrypted L2 links), [WebSockets-for-Python](https://github.com/Lawouach/WebSocket-for-Python)

```
server1$ sudo ./gelf.py --host 0.0.0.0 --enc 7001
[Gelf] Encryption key: b04c96f0e62fd14424e7e58e3d8ba957
[Gelf] Created interface: tap0
[Gelf] Relay listening: 0.0.0.0 7001
server1$ sudo ifconfig tap0 10.10.10.1 netmask 255.255.255.0
```

```
server2$ sudo ./gelf.py --host 0.0.0.0 --key b04c96f0e62fd14424e7e58e3d8ba957 7001
[Gelf] Encryption enabled
[Gelf] Created interface: tap0
[Gelf] Relay listening: 0.0.0.0 7001
server2$ sudo ifconfig tap0 10.10.10.2 netmask 255.255.255.0
```

Now comes the tricky part. Open a web browser and navigate to **server1**'s web server. This will automatically open a WebSocket to **server1**. Use the interface to add a WebSocket to **server2**.

![gelf-webclient-actions.png](/assets/images/gelf-webclient-actions.png)

```
server2$ ping -c3 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_req=1 ttl=64 time=9.17 ms
64 bytes from 10.10.10.1: icmp_req=2 ttl=64 time=3.50 ms
64 bytes from 10.10.10.1: icmp_req=3 ttl=64 time=3.83 ms

--- 10.10.10.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 3.504/5.505/9.175/2.599 ms
server2$ arp -an
[...]
? (10.10.10.1) at 4a:af:22:08:83:0f [ether] on tap0
```

How does it work? Also simple: On each server Gelf creates a _tun_ device, opens it for reading/writing, and forwards all communication to an HTML5 WebSocket. On the HTTP client a JavaScript routine manages multiple Gelf WebSockets and broadcasts data between them.

### Caveats

The version of WebKit in MobileSafari does not support RFC 6455. So the primary use case is defunct. However, RFC6455 is supported in the WebKit nightly build.

The JavaScript routine isn't clean. A JSON wrapper controls propagation of broadcasted messages. This is due to a potential for cyclic broadcast as two WebSocket clients are used per-host. One client is used in a poll/write thread, and another joined via HTTP client. A more elegant solution would remove the poll/write client and substitute a read directly from the WebSocket server thread.

Read buffers are not optimized. The configured read buffer should set the _tap_ interface's MTU. This should be optimized with respect to HTTP as a transport.

### Download / Clone / Fork

Feedback is certainly welcomed! [https://github.com/theopolis/gelf](https://github.com/theopolis/gelf)
