---
title: "SIM card curiosity, and a little Hardware Hacking"
created: 2011-12-12
categories: 
tags: 
  - hardware
  - diy
  - re
  - sim

---

A few months ago I took an interest in the layer 2/3 protocols (and their implementations) for mobile networks. I quickly arrived at SIM card hacking and like a young schoolboy thought, "man if only I could MitM the hardware communication I could spoof other's SIM cards and use free Internet!" Nope. Well, not nope, but it's not that easy.

<!--more-->

Let me try to follow my thought process here, bare with me please... The first question that happened in my brain was how to MitM SIM communication? Step 1: can I tap SIM card to modem communication? Luckily I had some artifacts from my experiments with UAVs, specifically a USBMA and several MiniPCIe modems. So I soldered a lone wire to the SIM I/O connector pin on the board. If you have the same curiosity, for a more universal fit, try an eBay search for "_SIM Card Extender_", and use that I/O connector pin.

Step 2: Google like crazy to find out if it's possible to read SIM I/O over serial (RS232). Why? I learned from [s7ephen's presentation](http://dontstuffbeansupyournose.com/2011/08/25/hardware-hacking-for-software-people/), most hardware protocols can be read over serial. Good news, SIM I/O outputs as a TTL protocol, which can be converted (from +0,1.8 to +3,5V) into serial (from -15,-5 to 5,15V) using a MAX232 integrated circuit (IC). Cool, but what the hell does that mean. Well, my hardware-savvy friends from UDel's CVORG research group gave me a little hand holding and pointed me towards a book and a beginning electronics’ kit. They assured me this was a very easy project, with little chance for failure. I'll show them little chance for failure!

If you're following along, here are some links:

- Tapping SIM card overview: [Wolfgang Rankl](http://www.wrankl.de/UThings/UThings.html#IFD2ICC) - Things for Smart Cards (some German)
- Make! [Make: Electronics Book](http://www.amazon.com/Make-Electronics-Discovery-Charles-Platt/dp/0596153740/)
- SparkFun: [Beginning Embedded Electronics](http://www.sparkfun.com/products/8373) - Power Supply Kit

I've become a huge fan (like everyone else) of SparkFun after my UAV experiments, so shopping for parts this time around was quite refreshing.

Ok back on track! The beginning electronics kit will provide a breadboard to prototype, a DIY power supply, and some hookup wire. However! Using any MAX232 wiring layout, but in this case check Wolfgang's site, you'll see we need a few more goodies. The MAX232 IC, 5× 1uF capacitors, and a female serial (RS232) adapter. Luckily you can also get these from SparkFun. Or you can purchase a MAX232 [kit](http://fcelectronics.mybisi.com/product/rs232-ttl-converteradapter-kit-max232-for-avrpicarmgps) (shop around though), but this is about learning, let's DIY! Add to the shopping cart:

- 1 [MAX232](http://www.sparkfun.com/products/589) IC
- 5 x [1uF/25V](http://www.sparkfun.com/products/8375) (Wolfgang says 1uF, they should work fine too) capacitors
- 9pin [RS232 female](http://www.sparkfun.com/products/429) mount

Follow the instructions on SparkFun's beginning electronics tutorial to build the power supply, and you should have all the knowledge needed to read the SIM to MAX32\-RS232 diagram on Wolfgang's site. All of this seemed really cool and new, but it's rather trivial when trying to hack with hardware. This setup only allows you to tap and possibly inject (if you add the RX wires) into the communication between a modem and SIM. We cannot break anything yet.

In the above image I'm using the breadboard from the beginning electronics’ kit from SparkFun, and on the left is the power supply I built while following their tutorial. In the center is my USBMA board with a 3G modem plugged in, and a wire extruding from the top attached to the SIM data I/O pin. This wire connects to the MAX323 TX input on the right, which converts the signal to be read from the RS232 on the far right. Note that the USBMA is powered and connected via USB to my PC, as well as the serial cable.

I'm using AT&T for my carrier, and passing the modem back and forth between VMs to elicit a firmware load and test connection. This allows me to create controls for understanding the communication between the modem and SIM card.

**Some SIM Hacking History**

SIM hacking isn't easy these days. Read [this](http://www.techtraction.com/2009/01/19/why-you-cannot-clone-a-sim-chip/) for a very defeatist point of view. The long story short, in order to fake a SIM card you need an ID (IMSI) and a secret key (Ki). The mobile provider must [authenticate](http://www.isaac.cs.berkeley.edu/isaac/gsm-faq.html) (darn) the IMSI by sending a RAND (random number) and confirming that the SIM can encrypt using the correct Ki. ((The provider knows both your IMSI and Ki because they originally wrote it to the card.))  This key generation / authentication (or hashing) algorithm is called A5/A8 which implement an algorithm called COMP128, which has multiple versions. On older SIM cards, using COMP128v1 (we're talking GSM pre-GPRS, pre-2002), the Ki could be extracted by brute forcing a [weakened](http://en.wikipedia.org/wiki/A5/1) 64->54bits algorithm. As far as my Google-fu allowed, there are no practical COMP128v2/3 attacks. The only promising article I found was by Rafzar, which [claims](http://www.rafzar.com/node/24) to have reverse engineered the micro-controller on the SIM using a combination of acid washes and logic analyzers.

Wait what!? There's an IC running ON the SIM card? No way!

Selected reading/watching:

- IASSC, Berkeley - [GSM Cloning](http://www.isaac.cs.berkeley.edu/isaac/gsm-faq.html) (article, includes crypto information)
- Charles Brookson - [How to Clone GSM SIM](assets/cardclone.pdf) (from 2005)
- Citizen Engineer - [SIM Card Hacking](http://citizenengineer.com/) (video)
- THC \- [SIM Toolkit Research Group](http://wiki.thc.org/gsm/simtoolkit) (Wiki)
- Ivan "e1" Buetler - [Smart Card APDU Analysis](assets/BH_US_08_Buetler_SmartCard_APDU_Analysis_V1_0_2.pdf) (Blackhat 2008)
- CCC \- [CCC will clone D2 card customers](http://dasalte.ccc.de/gsm/?language=en) (article, German)
- Billy Brumley - [COMP128 Walkthrough](assets/S5.Brumley-comp128.pdf) (class slides)
- Steve Hulton - [Intercepting GSM Traffic](assets/bh-dc-08-steve-dhulton.pdf) (Blackhat DC 2008)

**...to Ponder**

I was interested in spoofing another user's SIM to achieve mobile anonymity. However, a paid-in-cash AT&T GoPhone will mostly do the trick. It wont hide you location and it DOES cost money. Stealing another user's SIM will remove the money problem, but you wont be so quick to hide your location from the provider.

My next step is to add a MUXer so I can achieve a MitM to implement access control rules between the SIM and modem. The utility being: hiding or faking auxiliary data on the SIM. The reality being: I want to learn more! I'll end with some captures I took using pyserial:

```python
import serial, sys, os

class SIMtap(object):
    """customized class that accepts a serial port which is tapping a SIM-&amp;gt;Modem"""
    def \_\_init\_\_(self, dev):
        """Basic init method"""
        self.dev = dev
        # Open the serial device at SIM's clock of 3.75MHz
        self.\_\_serial = serial.Serial(dev, 9600, parity=serial.PARITY\_EVEN, rtscts=2)
        self.\_\_buffer = ""
        self.\_\_prevBuffer = ""
        if not self.\_\_serial.isOpen():
            print "\[SIMtap\]", "error opening dev", self.dev
            return 1
        
    def read(self):
        c = self.\_\_serial.read(1)
        self.\_\_buffer += c
        return c

def usage():
    print "%s: serial\_device" % sys.argv\[0\]
    print "\\twindows: COMx"
    print "\\tlinux: /dev/ttyXXX"
    
if \_\_name\_\_ == '\_\_main\_\_':
    if len(sys.argv) < 2:
        usage()
        exit(1)
    st = SIMtap(sys.argv\[1\])
    
    n = 0
    while os.path.exists("simio%s.bin" % str(n)):
        n += 1 
    if n &amp;gt; 0:
        prev= open("simio%s.bin" % str(n-1), 'r')
        st.\_\_prevBuffer= prev.readline() 
    output= open("simio%s.bin" % str(n), "w", 0)

    i = 0
    matching = ""
    while(1):
        i += 1
        if not i % 100:
            print ""
        c = st.read()
        output.write(c)
        output.flush()
        sys.stdout.write(c.encode("hex"))
        sys.stdout.flush()
        if c.encode("hex") == st.\_\_prevBuffer\[i-1\].encode("hex"):
            matching += c
            print "\\nMatched (%i bytes): %s" % (len(matching), matching.encode("hex"))
        else:
            matching = ""
    
    output.close()
```

Save contact list output:

```
a000f200fc00fd0000e0fe1000dafea000fafefd00ff0000800f406f3a040e1a0000f000fefb0f7fffffff1111111
111f1c000fdf800e0a41000dafea40000ef0000fd000030ff8b00440106f900fa00ff00ff000000e833c77011fe00
fd00f000fe5400991f00ff7ffcff7800ffe0f000f840f80084f800fc7800ff00fe00fd0400207ff8ff3c00ff3c00f
fff00ffff0f00ffff00f800ffff00ff00900000dc00c000ff00f800ff0f00ff0f00ffff00ffff0f00ff80f800ffff
00ff0000fc00f03f00ff7800ff3c00ffff00ffff1e00ff00f8f0ffff0f00ff000000ee008100ff00f8fff000fff00
0ff3c00ffff00ffffff00fffff8ff000d00ff00e000fff000ffff00ffff00f8ffff00ff00f800ff0f00ffff00ff00
fe00fddc001eff7f00ff00f8ffff00ff7800ffffff1e00ff0f00ff0008ffffffffffffffffffffffffffa020dcfff
fffffffffffffffffffffff0a04ffffffffffffffffffffffffffff9000dcebffffffffffffffffffffffffffffff
a020dcffffffffffffffffffff9000dc0dffffffffffffffffffffffffffa0dc20dcfffffffffffffffffffffffff
fff0ffcffffffffffffffffffffffffffffff90c0dcfcffffffffffffffffffffffffffff9000d4dcffffffffffff
ffffffffffffffa00420dcffffffffffffffffffffffffffff9000dc13ffffffffffffffffffffffffffa020dc...
```

In the above code I have a check for matching bytes to the previous capture because I suspected the line included noise. I did find some noise and this is on my "figure this out" hardware hacking to do list. After 6 captures, I found sections that remained constant. But I couldn't match the output to any APDU or AT commands. So I made the same I/O tap mod on a SIM reader, and captured the output of a known APDU command, issued by me. This worked correctly. I wonder what extra information the modem is sending? ((There are propriety AT and APDU commands.))
