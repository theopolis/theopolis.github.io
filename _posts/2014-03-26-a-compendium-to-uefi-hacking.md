---
title: "A Compendium to UEFI Hacking"
created: 2014-03-27
tags: 
  - firmware
  - uefi

image: /assets/images/img.jpg
---

There are quite a few operating/execution environments running below or before an Operating System's kernel. Computer science calls protection domains "Rings" and an Operating system's kernel is called "Ring 0" or "Supervisor mode". Researchers have called the lower-level environments Ring -1 (Hypervisor mode), and Ring -3 ("system management mode"), and they are fairly apt-names. I like to bundle all of these into a scary-but-funny-and-fitting name **_subzero_**, dun dun dun!

<!--more-->

![bios.jpg](/assets/images/uefi-hacking-bios.jpeg)

Intel and the UEFI (Universal Extensible Firmware Interface) forum embody a really awesome **_subzero_** concept highlighted in the UEFI acronym-expansion. That is, applying standards to highly-privileged protection domains allows software engineers and vendors to take advantage of each other's development and security improvements. Never-the-less, standards and their implementation-specific variations attract security researches too!

Over the last few years there have been various vulnerabilities, exploitation, malware proof-of-concepts, attach strategies, frameworks, and research topics related to the new **_subzero_** developments. Like a kid in a hobby shop, I've become enthused by this world and cannot stop reading about what these epic researchers have accomplished. This article pays homage to this community by trying to compile the related work.

This document/chrono-log is no where near complete. It would be very helpful if mistakes, discrepancies, and in-adequateness could be emailed to "teddy@prosauce.org". I'll try to apply changes and additions ASAP. Keep in mind I cite work very similar to hundreds of derived work and concepts. My goal is to capture seminal, influential and ground-breaking concepts. And to this goal I welcome any criticism.

This will include offensive research related to: BIOS (though only more-recent BIOS research) because BIOS and BIOS Plug and Play concepts are still used; virtualization concepts as they apply to Ring 0 subversion; rootkits and bootkits that take advances of **_subzero_** concepts and technologies; ACPI (Advanced Configuration and Power Interface(s)); PCI OptionROM; UEFI; System Management Mode, Intel TXT and VT-d; Intel AMT and the Intel Management Engine. I'll loosely try to create several categories listed below:

### **Firmware, BIOS, UEFI Rootkits, Malware, and Offensive Concepts:**

[ACPI BIOS Rootkit](assets/bh-eu-06-Heasman.pdf)  
by: John Heasman, NGS Software  
published: 2006 at BlackHat Europe

[PCI OptionRom Rootkit](assets/bh-dc-07-Heasman-WP.pdf)  
by: John Heasman, NGS Software  
published: 2007 at BlackHat DC

[Your computer is now stoned (...again!)](assets/Kasslin-Florio-VB2008.pdf)  
by: Kimmo Kasslin and Elia Florio, F-Secure and Symantec  
published: 2008 as an analysis of the Mebroot-family (Mebromi) of MBR-based Rootkits

[SMM Rootkits: A New Breed of OS Independent Malware](assets/SMM-Rootkits-Securecom08.pdf)  
by: Shawn Embleton, Sherri Sparks, Cliff Zou  
published: 2008 at SecureCom

[BIOS-level Windows Rootkit, and Persistent BIOS Infection](assets/csw09-sacco-ortega.pdf)  
by: Alfredo Ortega and Anibal Sacco, Core Security Technologies  
published: 2009 at CanSecWest Conference  
additional references: [Reactivate the Rootkit: Attacks on BIOS anti-theft technologies](assets/BHUSA09-Ortega-DeactivateRootkit-PAPER.pdf)  
by: Alfredo Ortega, Anibal Sacco, Core Security Technologies  
published: 2009 at BlackHat USA

[Hardware Backdooring is practical, and Rakshasa](assets/BH_US_12_Brossard_Backdoor_Hacking_Slides.pdf)  
by: Jonathan Brossard, Toucan System  
published: 2012 BlackHat USA

[De Mysteriis Dom Jobsivs: Mac EFI Rootkits](assets/De_Mysteriis_Dom_Jobsivs_-_Syscan.pdf)  
by: Loukas K (snare)  
published: 2012 at SyScan Singapore  
additional references: [BlackHat 2012 USA Whitepaper](assets/De_Mysteriis_Dom_Jobsivs_Black_Hat_Paper.pdf)

[When Firmware Modifications Attack: Embedded Exploitation](assets/ndss-2013.pdf)  
by: Ang Cui, Michael Costello, and Salvatore Stolfo  
published: 2013 at NDSS

[Implementation and Implications of a Stealth Hard-Drive Backdoor](assets/acsac13.pdf)  
by: Jonas Zaddach, Anil Kurmus, Davide Balzarotti, Erik-Oliver Blass, Aurelien Francillon, Travis Goodspeed, Moitrayee Gupta, and Ioannis Koltsidas  
published: 2013 at ACSAC

### **Virtualization-focused Exploits and Rootkits:**

[SubVert: Implementating malware with virtual machines](assets/SubVirt%3A%20Implementing%20malware%20with%20virtual%20machines.pdf)  
by: Samuel T. King and Peter M. Chen  
published: 2006 at IEEE Security and Privacy (Oakland)

[Bluepill: Subverting Vista Kernel for Fun and Profit](assets/BH-US-06-Rutkowska.pdf)  
by: Joanna Rutkowska, Advanced Malware Labs, COSEINC  
published: 2006 SyScan Conference in Singapore  
additional references: [IsGameOver()](assets/IsGameOver.pdf)

[Hardware Virtualization Rootkits](assets/BH-US-06-Zovi.pdf)  
by: Dino A. Dai Zovi  
published: 2006 at BlackHat USA

[VBoot Kit: Compromising Windows Vista Security](assets/bh-eu-07-kumar-apr19.pdf)  
by: Nitin Kumar and Vipin Kumar, NVLabs  
published: 2007 at BlackHat Europe

[BIOS Boot Hijacking and VMware Vulnerabilities Digging](assets/sunbing.pdf)  
by: Sun Bing  
published: 2007 at POC, Seoul Korea

### **Firmware, BIOS, and UEFI Vulnerabilities and Exploitation:**

[System Management Mode to Circumvent Operating System Security](assets/duflot.pdf)  
by: Loic Duflot, Daniel Etiemble, and Olivier Grumelard  
published: 2006 at CanSecWest Conference

[Hacking the Extensible Firmware Interface](assets/bh-usa-07-heasman.pdf)  
by: John Heasman, NGS Software  
published: 2007 at BlackHat USA

[Attacking Intel Trusted Execution Technology](assets/Attacking%20Intel%20TXT%20-%20slides.pdf)  
by: Rafal Wojtczul and Joanna Rutkowska  
published: 2009 at BlackHat DC

[Attacking SMM Memory via Intel CPU Cache Poisoning](assets/smm_cache_fun.pdf)  
by: Rafal Wojtczuk and Joanna Rutkowska  
published: 2009 by Invisible Things Labs

[Attacking Intel BIOS](assets/BHUSA09-Wojtczuk-AtkIntelBios-SLIDES.pdf)  
by: Rafal Wojtczuk and Alexander Tereshkin  
published: 2009 at BlackHat USA

[Introducing Ring -3 Rootkits](assets/Ring%20-3%20Rootkits.pdf)  
by: Alexander Tereshkin and Rafal Wojtczuk  
published: 2009 at BlackHat USA  
additional references: [Another Way to Circumvent Intel Trusted Execution Technology](assets/Another%20TXT%20Attack.pdf)  
by: Rafal Wojtczuk, Joanna Rutkowska, and Alexander Tereshkin  
published: 2009 by Invisible Things Labs

[Following the White Rabbit: Software Attacks against Intel VT-d](assets/Software%20Attacks%20on%20Intel%20VT-d.pdf)  
by: Rafal Wojtczuk and Joanna Rutkowska  
published: 2011 by Invisible Things Labs

[Exploring new lands on Intel CPUs](assets/Attacking_Intel_TXT_via_SINIT_hijacking.pdf) (SINIT code execution hijacking)  
by: Rafal Wojtczuk and Joanna Rutkowska  
published: 2011 by Invisible Things Labs

[BIOS Chronomancy: Fixing the Core Root of Trust for Measurement](assets/US-13-Butterworth-BIOS-Security-Slides.pdf)  
by: John Butterworth, Corey Kallenberg, Xeno Kovah, The MITRE Corporation  
published: 2013 at BlackHat USA

**(Coming Soon): All Your Boot are Belong to Us**  
by: Corey Kallenberg, Yuriy Bulygin, Andrew Furtak, Oleksandr Bazhaniuk, John Loucaides, Xeno Kovah, John Butterworth, Sam Cornwell, Intel and MITRE  
published: 2014 at CanSecWest Conference

### **Related Vulnerability Advisories:**

**Intel-SA-00017: (2008) BIOS SMM Privilege Escalation**  
https://security-center.intel.com/advisory.aspx?intelid=INTEL-SA-00017&languageid=en-fr

**Asus EEE PC: (2009) BIOS SMM Privilege Escalation Vulnerabilities**  
http://archives.neohapsis.com/archives/bugtraq/2009-08/0059.html

**Intel-SA-00018: (2009) BIOS SMM Privilege Escalation**  
https://security-center.intel.com/advisory.aspx?intelid=INTEL-SA-00018&languageid=en-fr

**Intel-SA-00019: (2009) Unauthorized Downgrading to a previous BIOS version**  
https://security-center.intel.com/advisory.aspx?intelid=INTEL-SA-00019&languageid=en-fr

**Intel-SA-00020: (2009) Buffer Overflow Local Privilege Escalation**  
https://security-center.intel.com/advisory.aspx?intelid=INTEL-SA-00020&languageid=en-fr

**Intel-SA-00021: (2009) SINIT Misconfiguration allows for Privilege Escalation**  
https://security-center.intel.com/advisory.aspx?intelid=INTEL-SA-00021&languageid=en-fr

**Intel-SA-00022: (2010) BIOS SMM Privilege Escalation**  
https://security-center.intel.com/advisory.aspx?intelid=INTEL-SA-00022&languageid=en-fr

**Intel-SA-20023: (2010) Intel AMT Software Development Kit Remote Code Execution**  
https://security-center.intel.com/advisory.aspx?intelid=INTEL-SA-00023&languageid=en-fr

**Intel-SA-00030: (2011) SINIT Buffer Overflow Vulnerability (example)**  
https://security-center.intel.com/advisory.aspx?intelid=INTEL-SA-00030&languageid=en-fr

**CVE-2011-1898: VT-d (PCI passthrough) MSI**  
http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2011-1898

**VU#912156: (2013) Dell BIOS RBU Packet Buffer Overflow**  
http://www.kb.cert.org/vuls/id/912156

### **Postamble:**

This list is a massive work in-progress. I hope to augment it with **_subzero_** defenses and related security tools very soon!
