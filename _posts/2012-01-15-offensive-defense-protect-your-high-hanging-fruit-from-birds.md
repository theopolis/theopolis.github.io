---
title: "Offensive Defense: Protect your high-hanging fruit ...from birds and stuff"
created: 2012-01-15
tags: 
  - defensive
  - offensive
  - fun
---

One of these days this webserver will be torn open by some low-hanging vulnerability. Sure, but that wont be very exciting, so let's think outside of the inevitable, and into the what-if.

<!--more-->

What-if someone did break into this poor little webserver? Regardless of how they did it, what would they do? What would they find? Step 1: Break into my box, Step 2: ..., Step 3: Profit. You'll achieve profit without any 'Step 2' by killing my ego and any minuscule reputation I have among my friends. But assuming you're not out for defamation: let's think about the 'Step 2', and some possible defensive methods to protect a box once someone has broken in.

I call these techniques offensive-defense. They are techniques designed to provide defense once an attacker has gained access. In this case, you are not expecting them to gain access, but you are providing an additional layer of hardship incase the unexpected occurs. Note: this is not the same as turning your box into a honeypot. We are not interested in what the attacker is after, nor are we inviting attackers; we are concerned with scaring them away! And as always, this is a fun exercise! I've not tested any techniques against a real attacker, and I will not guarantee they enhance security, they are merely POCs.

### **1\. Replace common binaries with a lock-out script**

This is my favorite defense-after-compromise technique. Choose carefully the binaries you normal system-automation scripts don't use and move them aside. To be even more sly, replace them! If you know they shouldn't be run without your conscience decision to do so, make them log the uninitiated user off your box!

Here's an example script that will help you manage your "do not run me" binaries. Using the below code, issue a 'remove' to backup all identified binaries to a location of your choosing, then replace them with a hard link to the lock-out script. Issue a 'restore' to move all your backed-up binaries so you can effectively use your system again.

```python
import sys,os,subprocess,shutil

### Set the binaries you want to protect
_PROGS_ = ['nc']#, ['wget', '/usr/local/bin'], 'ssh']
### Where should they be backed up?
_BACKUPS_ = “/usr/local/tbin”
### For bins without an included path
_DEFPATH_ = “/usr/bin”

def path(prog): return prog[1] if type(prog) == list else _DEFPATH_
def bin(prog): return prog[0] if type(prog) == list else prog

def error(err):
	print “Warning: %s” % err
	#exit(2)

### Create the logout program, in this case, logout the shell
### Could replace with a python script using getpass.getuser()
### to log out the current user
def create_bastard():
	bs = open(“%s/bastard” % _BACKUPS_, 'w')
	bs.write(“““#!/bin/sh\n\nkill -HUP `pgrep -s 0 -o`”””)
	bs.close()
	os.chmod(“%s/bastard” % _BACKUPS_, 0755)

def main(args):
	remove() if args[0] == 'remove' else restore()

def restore():
	'''Create the correct hard links to backup bins'''
	for prog in _PROGS_:
		if os.path.exists(“%s/%s” % (_BACKUPS_, bin(prog))):
			try: os.link(“%s/%s” % (_BACKUPS_, bin(prog)), “%s/%s” % (path(prog), bin(prog)))
			except OSError as err: error(err)
			
def remove():
	'''Remove hard links if they exist, and replace with fake'''
	for prog in _PROGS_:
		p = “%s/%s” % (path(prog), bin(prog))
		if os.path.exists(p): 
			### Not the best way to check, but works for now
			### Replace with a hash for better results
			nl = os.stat(p).st_nlink
			if nl == 2:
				#if not raw_input(“Removing fake copy of %s [Y/n]” % bin(prog)) == 'n':
				print “Removing real copy of %s.” % bin(prog)
				try: os.remove(p)
				except OSError as err: error(err)
			elif nl == 1: 
				if os.path.exists(“%s/%s” % (_BACKUPS_, bin(prog))):
					try: os.remove(p) 
					except OSError as err: error(err)
				if not raw_input(“Move real copy of %s to %s [Y/n] “ % (bin(prog), _BACKUPS_)) == 'n':
					try: shutil.move(p, _BACKUPS_)
					except: error(sys.exc_info()[1])
		nl = os.stat(p).st_nlink if os.path.exists(p) else 0
		if nl and nl == len(_PROGS_) + 1:
			print “Hard link of %s already points to the fake.” % bin(prog)
			continue
		if not os.path.exists(“%s/bastard” % _BACKUPS_):
			create_bastard()
		print “Linking %s/bastard to %s/%s” % (_BACKUPS_, path(prog), bin(prog))
		try: os.link(“%s/bastard” % _BACKUPS_, “%s/%s” % (path(prog), bin(prog)))
		except OSError as err: error(err)
		
if __name__ == '__main__':
	if len(sys.argv) < 2 or not sys.argv[1] in ('remove', 'restore'):
		print “Usage: %s [remove|restore]” % sys.argv[0]
		print “  Affecting: %s” % “, “.join(map(lambda a: bin(a), _PROGS_))
		exit(1)
	if not os.path.exists(_BACKUPS_):
		if not raw_input(“Warning: %s does not exist, should we create? [Y/n] “ % _BACKUPS_) == 'n':
			try: os.makedirs(_BACKUPS_)
			except OSError as err: error(err); exit(2)
	main(sys.argv[1:])
```

### **2\. Require interactive sessions to disarm a kick-out script**

If it's not an inconvenience to type a disarm command each time you open an interactive session, then consider something related to the script below. If an uninitiated-user opens an interactive session, they'll quickly be warned and kicked out. Follow-up with a script that blocks traffic from the offending host (this could be really annoying if you forgot to disarm your own shell, so I'd recommend whitelisting your host).

Spawn this script when a shell is opened, as an example.

```c
#include <stdio.h>
#include <strings.h>
#include <stdlib.h>
#include <pwd.h>
#include <unistd.h>

#define READ 0
#define WRITE 1
/* MGS = I'm watching you... */
#define MSG {73, 39, 109, 32, 119, 97, 116, 99, 104, 105, 110, 103, 32, 121, 111, 117, 46, 46, 46, 0}

/* Bi-directional popen: http://snippets.dzone.com/posts/show/1134 */
pid_t popen2(const char *command, int *infp, int *outfp) {
    int p_stdin[2], p_stdout[2];
    pid_t pid;
    
    if (pipe(p_stdin) != 0 || pipe(p_stdout) != 0)
        return -1;
    
    pid = fork();
    if (pid < 0)
        return pid;
    else if (pid == 0) {
        close(p_stdin[WRITE]);
        dup2(p_stdin[READ], READ);
        close(p_stdout[READ]);
        dup2(p_stdout[WRITE], WRITE);
        
        execl(“/bin/bash”, “sh”, “-c”, command, NULL);
        perror(“execl”);
        exit(1);
    }
    
    if (infp == NULL) close(p_stdin[WRITE]);
    else *infp = p_stdin[WRITE];
    if (outfp == NULL) close(p_stdout[READ]);
    else *outfp = p_stdout[READ];
    return pid;
}

void t_getuser(char *user) {
    struct passwd *pw;
    uid_t uid;
    
    uid = geteuid();
    pw = getpwuid(uid);
    if (pw)
        snprintf(user, 128, “%s”, pw->pw_name);
    else {
        perror(“getpwuid”);
        exit(1);
    }
    return;
}

int main(int argc, char *argv[]) {
    /* Loc of write */
    char *call = “/usr/bin/write”;
    char *call2 = “/usr/bin/pkill”;
    char pipe_word[20] = MSG;
    struct passwd *pw;
    uid_t uid;
    char *user;
    char cmd[128];

    /* Guessing 128-24 is a good max username size, larger usernames wont work :( */
    user = malloc(128);
    bzero((void *) user, 128);
    t_getuser(user);

    bzero((void *) cmd, 128);
    snprintf(cmd, 128, “%s %s”, call, user);

    int infp, outfp;
    
    /* Open write */
    if (popen2(cmd, &infp, &outfp) <= 0) {
        perror(“popen2”);
        exit(1);
    }
    
    /* Write message as stdin */
    write(infp, pipe_word, 19);
    write(infp, “\n”, 1);
    close(infp);
    close(outfp);
    sleep(1);
    
    /* Log out the user */
    bzero((void *) cmd, 128);
    snprintf(cmd, 128, “%s -KILL -u %s”, call2, user);
    execl(“/bin/bash”, “sh”, “-c”, cmd, NULL);
    return 0;
}
```

### **3\. Install a key-logger**

Install a keylogger, like [logkeys](http://code.google.com/p/logkeys/), then create an canary that watches the key log file and emails a digest upon activity. If someone does break into your box, at least you know what they did afterward. Hopefully they'll drop credentials, a callback host, or some other identifying piece of information.

### **4\. Create a "scary" shell login message**

Simple. Just add something that might frighten an attacker from using your box. Make sure you're not taunting them. Something related to: _"Secret Government Server, do not use under penalty of death."_ May not have the desired effect. I prefer calling the box a honeypot, an extremely low-value target meant to have been broken. Hell, the attacker might give you some cred, thinking you installed another layer of defense around your real box. Just add your message to _/etc/motd_. In this example it looks like I just installed some honeypot software, and forgot to configure it properly.

“Welcome to \[hostname\] honeypot. Please remove this message from (/etc/motd) before starting the honeypot daemon.”

### **5\. Run a fake local DNS**

This idea really depends on your setup. I profiled my small webserver for half a week and saw only DNS PTR requests. So if you filtered all A requests to a local/fake DNS server you may cause an awesome amount of frustration for an attacker. That is until they point your box to an attacker-controller or public DNS resolver.

I pulled a [fake DNS](http://code.activestate.com/recipes/491264-mini-fake-dns-server/) server from Francisco Santos, which is included on the [REMnux](http://remnux.org/) Linux Distro. Then made a small modification to forward all non-A requests to real DNS server.

```python
import socket

FAKE_IP = '192.168.1.1'
FORWARDER = '8.8.4.4'

class DNSQuery:
  def __init__(self, data):
    self.data=data
    self.domain=''
    self.type=0

    tipo = (ord(data[2]) >> 3) & 15   # Opcode bits
    if tipo == 0:                     # Standard query
      ini=12
      lon=ord(data[ini])
      while lon != 0:
        self.domain+=data[ini+1:ini+lon+1]+'.'
        ini+=lon+1
        lon=ord(data[ini])
      if data[ini+1:ini+3] == '\x00\x01':
        self.type='A'

  def response(self, ip):
    packet=''
    if self.domain:
      packet+=self.data[:2] + '\x81\x80'
      packet+=self.data[4:6] + self.data[4:6] + '\x00\x00\x00\x00'   # Questions and Answers Counts
      packet+=self.data[12:]                                         # Original Domain Name Question
      packet+='\xc0\x0c'                                             # Pointer to domain name
      packet+='\x00\x01\x00\x01\x00\x00\x00\x3c\x00\x04'             # Response type, ttl and resource data length -> 4 bytes
      packet+=str.join('',map(lambda x: chr(int(x)), ip.split('.'))) # 4bytes of IP
    return packet

if __name__ == '__main__':
  print 'pyfakeDNS:: dom.query. 60 IN A %s' % FAKE_IP
  
  udps = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
  udps.bind(('',53))
  
  try:
    while 1:
      data, addr = udps.recvfrom(1024)
      p=DNSQuery(data)

      if p.type == 'A':
        udps.sendto(p.response(FAKE_IP), addr)
      	print 'Response: %s -> %s' % (p.domain, FAKE_IP)
      else:
      	forwards = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
      	forwards.connect((FORWARDER, 53))
      	forwards.send(data)
      	data = forwards.recv(1024)
      	forwards.close()
      	udps.sendto(data, addr)
      	print 'Forwarded non-A request'

  except KeyboardInterrupt:
    print 'Finalizando'
    udps.close()
```

### **6\. Rename all processes**

Or... just include a 'modified' version of libproc. Download procps-3.2.8, apply your distribution patches... and then this extra patch. Move your real proc utilities aside, and replace them with fake versions linked against the 'modified' libproc. This will cause all commands listed by ps to say '---hackers no hacking---'.

```c
--- procps-3.2.8~/proc/escape.c 2005-01-05 15:50:26.000000000 -0500
+++ procps-3.2.8/proc/escape.c 2012-01-21 01:30:12.946562284 -0500
@@ -182,7 +182,7 @@

if(flags & ESC_ARGS){
const char **lc = (const char**)pp->cmdline;
- if(lc && *lc) return escape_strlist(outbuf, lc, bytes, cells);
+ //if(lc && *lc) return escape_strlist(outbuf, lc, bytes, cells);
}
if(flags & ESC_BRACKETS){
overhead += 2;
@@ -201,7 +201,8 @@
outbuf[end++] = '[';
}
*cells -= overhead;
- end += escape_str(outbuf+end, pp->cmd, bytes-overhead, cells);
+ char *fake_cmd = “---hackers no hacking!---”;
+ end += escape_str(outbuf+end, fake_cmd, bytes-overhead, cells);

// Hmmm, do we want “[foo] “ or “[foo ]”?
if(flags & ESC_BRACKETS){
```

Example: compile the patched version of procps to obtain a libproc-3.2.8.so\[-fake\] and ps\[-fake\]. Then create a wrapper for ps, which will be copied to /bin/ps:

```bash
#!/bin/sh
LD_LIBRARY_PATH=/opt/proc exec /bin/fps “$@”
mv /bin/ps /bin/rps
cp libproc-3.2.8.so-fake /opt/proc/libproc-3.2.8.so
cp ps-wrapper.sh /bin/ps
cp ps-fake /bin/fps
```

Now if you want a real listing of processes you can run /bin/rps. Otherwise a ps will output something a little less real.

```bash
teddy@ubuntu:~/offensive-def/procps-3.2.8/ps$ ./p aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
[...]
teddy    17793  0.0  0.3   7176  3900 pts/2    Ss   12:39   0:00 [---hackers no hacking!---]
teddy    18962  0.0  0.0   3880   504 ?        S    14:53   0:00 [---hackers no hacking!---]
teddy    18974  0.0  0.1   4572  1076 pts/2    R+   14:54   0:00 [---hackers no hacking!---]
```

### **7\. Delay egress connections**

Why delay egress connections? I'd argue- most of the time once you have access on a box you'll want to install a persistence mechanism, elevate your privileges, use recon tools, or exfil data. These steps \[most of the time\] require the attacker to pull down binaries or source code onto the box.

This largely depends on what type of services you box is running, and definitely how you use the box. In my case, the box I experimented on was just a webserver that allowed interactive SSH logins. Since there were no unsolicited TCP connections (aside from package/OS updates) delaying SYN packets by 5 seconds was a reasonable tactic. Imagine an attacker experiencing 5 seconds of delay on their TCP connections, while their pings are reporting otherwise. 5 seconds too soon? Try something more annoying.

```bash
ipfw pipe 400 config delay 5000
ipfw add 400 pipe 400 tcp from me to not me out tcpflags syn
```

### **8\. Custom software versions**

I've seen plenty of howtos that suggest removing the software version from banners, error pages, and the like. How about, instead of removing the version, you change it to something weird. Make your Apache server say it's nginx, change the version of SSH to something that'll never be released. If you view the source of this article \[on my ProSauce domain\] you'll see I'm running the new-and-improved version of Wordpress, 4.0 baby. Of course there are plenty of ways to heuristically detect the software version, but I'm going to make sure I do my share of fake advertisements.

And don't stop with external services, for your kernel: _\--append-to-version=.030309_. If someone does break into your box as a limited user, make sure they earn those escalated privileges.

### **9\. Mount a fake boot partition**

You can do this any way you feel comfortable. Essentially, this may force the attacker to exfil the wrong kernel binary and waste their time trying to locate additional vulnerabilities \[in your kernel\]. You could also save copies of kernels from other operating systems (OSX, FreeBSD, etc.) to create more confusion.

### **10\. Create a tripwire process**

Not a very effective tactic, but something I've found useful. Imagine an attacker performing recon on the box and noticing a process called _scan-for-hackers.sh_. Might make them a bit jumpy? You can take two routes here, either make the process respawn like a salmon in heat (thinking kernel module or something similar) or cause it to relaunch and logout the user.

_And that's all I've got as of now. If you can think of anything else maliciously defensive, let me know and I'll append!_
