---
title: "Adventures in UAV Hacking"
created: 2011-11-22
tags: 
  - uav
  - botnets
  - conferences
  - vip

---

My first accepted workshop paper, accepted to USENIX WOOT 2011, was called "SkyNET: A 3G-enabled mobile attack drone and stealth botmaster". Catchy name, right? Check out the [project page](http://www.cs.stevens.edu/~spock/skynet/ "SkyNET Project Page") if you'd like a review. After the paper was published, presented, and let lie for a month, the project caught the attention of MIT Technology Review. Shortly after the story was published tons of other websites started duplicating and running their own. The relation between UAVs and "Skynet" did the trick in attracting media attention. Unfortunately there's very little AI incorporated thus far into the project. Nevertheless, it's been a blast reading the various comments on the project.

<!--more-->

The goal of the project was to demonstrate a simple idea: use kinetic components to causes digital malevolence. We focused on botnets, combined with an interest in mobile and wireless technology; and decided to improve botnets using a UAV. Simple: UAV \+ botnet = better botnet. Add some sciencey details related to feasibility, an example command and control architecture, and call it a wrap! Well not quite, in order to comment on feasibility we need to know how long a UAV could fly, if auto-pilot were possibly, and most importantly, how much money a botmaster needs to fork over. There are existing UAVs designed for hacking and cracking, such as the [WASP](http://rabbit-hole.org/ "WASP"), but we needed something less expensive, and more available to the everyday botmaster. Opportunity!

"Hi professor, we need to purchase and build a UAV botmaster" "Sure!" And thus, one of the most fun senior projects ((in my opinion)) was created.

I'd like to take another opportunity and discuss some of the systems and technical parts of SkyNET. In story form of course!

**Hardware Design**

My primary task was to build, test, and provide feasibility details for the drone. To start, the group brainstormed requirements:

1. UAV
2. Linux kernel
3. 3G modem
4. At least one Wi-fi radio

We were computer scientists with little hardware hacking experience, we wanted to write code, and plug things in. With a little research we decided on an [AR.Drone](http://ardrone.parrot.com/parrot-ar-drone/usa/ "AR.Drone") to achieve UAV status, and a [TS-7552](http://www.embeddedarm.com/products/board-detail.php?product=TS-7552 "TS-7552") running Debian Lenny (ARM), to run our code. The TS-7552 single board computer was the only one we could find which included an onboard Mini-PCIe slot. This project was a function of cost, for both our wallets, and the feasibility. Mini-PCIe cards were the cheapest way to a 3G modem, and there was already one in my laptop.

**Current cost to botmaster:** $300 (AR.Drone) + $170 (TS-7552) + $30 (Mini-PCIe modem, via fast eBay search) + $20 (Mini-SD for storage) + $20 (Wi-fi radio)  
**Current work for botmaster:** Unknown at this point...

I mentioned feasibility was a function of time and money, UAV flight time was a function of weight and power. The AR.Drone was [tested](http://www.ardrone-flyers.com/forum/viewtopic.php?f=7&t=38 "AR.Drone Flyers forum testing weight") to hold a maximum payload of about 250g. By adding the TS-7552, 3G modem, radio, harness, and required wiring to power the TS-7552 then position the radios to balance the weight-- we added about 170g. At this point I could not get the drone to fly straight. I performed weight tests myself and found that the drone would become very unstable after 154g of payload. I proceeded under the assumption that 20g of payload could be removed easily by desoldering some of the TS-7552 components, and finding lighter radios in the future. Fortunately friends at Defcon's HHV removed 24g from the TS-7552, included above is a before-and-after.

The added payload decreases maximum flight time, but we regained a bit by substituting the stock 1000Mah battery with a 2g lighter aftermarket-1300Mah battery purchased for $21 on eBay. It seems the price now is from $20 to $45. The 1300Mah batteries fit into the stock battery compartment on the AR.Drone. More powerful batteries are available, but take up much more space.

The AR.Drone indoor hull was modified to fit the TS-7552 snug against the battery. Then I added two antenna for the 3G modem to the top, cut a space in the rear for the Wi-fi radio, and painted it black to add the desired malevolent effect. ;) Note: search eBay for internal Wi-fi antenna, I purchased mine for $5.50 after shipping.

To complete the hardware design, I spliced the power cable on the AR.Drone motherboard to provide power to the TS-7552. This was the most difficult task as it requires disassembly of the AR.Drone.

**Current cost:** $540 + $2.50 (power cable) + \[$21 (more powerful battery)\] + \[$5.50 3G antenna\]  
**Current work:** Disassemble AR.Drone, cut and crimp a Y-power adapter; Cut AR.Drone indoor hull with scissors, tape Wi-fi card to rear.

Finally to balance the contraption I bought 4 kitchen scales from target. For completeness I added a box of sandwich bags. ;)

**Software Design**

The best [part](http://www.ardrone-flyers.com/wiki/Technical_Specification "AR.Drone Tech-specs") of using an AR.Drone is the auto-telemetry. Using one AT command, this thing will hover in-place at waist height. To pilot the UAV, a user connects to an ad-hoc network broadcasted by the drone with their mobile phone. SkyNET requires the botmaster to pilot the drone via the Internet, as well as auto-pilot. To pilot SkyNET manually a custom Java applet was written to proxy AT commands. The TS-7552 connected to the drone's ad-hoc network, as well as an OpenVPN to an arbitrary host on the Internet, then waited for AT commands from that host, and forwarded them to the drone. Thus using the Java applet, a botmaster could point their AT commands to the OpenVPN host, and they would arrive at the drone via the TS-7552.

The AR.Drone includes an [SDK](http://projects.ardrone.org/wiki/ardrone-api/Multiplatform_examples "AR.Drone SDK") for creating awesome multi-platform interfaces. And I'll include a demo Java applet at the end, which will connect to the AR.Drone's [video stream](http://projects.ardrone.org/boards/1/topics/show/1724 "Video stream decoding"). Adding flight control will be an exercise left to the reader. This is actually very easy, follow the examples in the SDK. I'm not including the full code because I'm still working on the project, and I don't feel like pruning the code for new-and-exciting features not-yet-released!

That said, here's a list of software requirements:

1. GPS logging
2. AR.Drone connection maintenance
3. 3G connection maintenance
4. OpenVPN proxy
5. Cloud-based WPA, WPA2, WEP cracking interface
6. Various network and system attacking applications
7. Botnet command and control

I've not mentioned GPS until this point because the functionality came with the 3G modem we choose. I'll not list the exact model but there are plenty on eBay (both USB and Mini-PCIe) which support GPS. On Linux this is exposed to the OS using various 3G modem drivers. My modem used a Qualcomm Gobi 2000 chip, the documentation for this exists in various [places](http://www.thinkwiki.org/wiki/Qualcomm_Gobi_2000 "Qualcomm Gobi 2000 Linux"). Does Debian Lenny (ARM) include [gpsd](http://packages.debian.org/lenny/arm/gpsd/download "Debian Lenny (ARM) gpsd download"), yes. Logging GPS location is simple using [gpslogger](http://man.cx/cgps(1) "man cgps").

  
How all the components fit togetherTo connect to the AR.Drone we place a script which polls for a configured BSSID every 2 seconds. Not very smart, I've since improved this to connect via the AR.Drone's USB gadget interface. To use the USB interface you'll need to compile your own AR.Drone kernel, create or purchase an AR.Drone USB cable, then find or write a program to talk AT commands via the interface. Start [here](http://embedded-software.blogspot.com/2011/01/creating-testing-and-flashing-tbd.html "Flashing custom AR.Drone kernels"), download [these](http://code.google.com/p/ardrone-tool/ "AR.Drone plf tools"), read [this](http://www.rcgroups.com/forums/showthread.php?t=1335257&page=33 "RC Controlled AR.Drone"), and you'll be on your way. Woah!? You can compile your own AR.Drone kernel? Yes! Parrot provides both the source and a cross compiling tool chain, simply create an account on their Redmine site.

We choose AT&T for our 3G provider because we could purchase a data plan with cash, no name, address, email, or credit card (called a GoPhone plan). This was essential to creating a feasible mobile attack drone. Connecting to AT&T is simple in Linux, there are tons of [examples](http://wiki.inisec.com/index.php/Att_3G_on_Ubuntu "AT&T 3G on Ubuntu") on how to configure your PPP peer, and how to 'chat' with AT&T. You'll need a chat script custom to your provider and modem (in some cases modems support custom AT commands). From my experience, it's best to put your GoPhone SIM in a store model phone and send an initial text message to 'activate' the plan. I'm not sure if this is required, but it did the trick for me, and I've seen it recommended a few times. Using the PPP event scripts we configured the TS-7552 to establish an OpenVPN link as soon as the modem received a valid IP.

I wont provide any details about the cloud-based cracking interface other than it used EC2 and was mainly written in python. SkyNET was used to off-load CPU intensive tasks, which are not feasible on the TS-7552. The various network and system attacking applications are generic, you can imagine the types of tools which would be handy on a mobile attack drone. And if you're interested in the botnet command and control protocol example, check out the USENIX WOOT 2011 [paper](assets/Reed.pdf "SkyNET Paper").

**Epilogue**

I highly recommend the AR.Drone to those wanting to demonstrate UAV capabilities with a twist. It's not the stablest thing in the world, but it will let you dive head first into code without worrying about mechanics. There's a great community filled with very talented programmers, engineers, and thinkers. Searching the various [forums](http://www.ardrone-flyers.com/ "AR.Drone Flyers") will turn up nuggets on attaching 3G modems, GPS devices, additional cameras, themed hulls, and more! Definitely check it out!

**Current Cost:** $542.50 + \[OpenVPN Proxy cost\] + \[EC2 bill\]  
**Current Work:** That described above, plus the code.

If you read the paper you'll understand why the OpenVPN cost is negligible (it's only used once), and the EC2 bill wont be high, heck if you're a botmaster you'll be charging this whole ordeal to a stolen credit card anyways...
