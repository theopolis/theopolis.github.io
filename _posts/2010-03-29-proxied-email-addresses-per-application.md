---
title: "Proxied Email Addresses per-Application"
created: 2010-03-30
tags: 
  - email
  - vip

---

Abstract: I wanted some mechanic in email, such that I could tell how a person found my address. Since I post my email address to many web sites, I wanted a way to track which web sites knew which emails. So I wrote a small script which allowed me to think of email addresses on the fly and distribute them. When someone responded to the emails they would be analyzed and forwarded to my inbox with a reply-to address that would take the opposite route and be sent as the 'on-the-fly' email address.

<!--more-->

Problem: There are two problems. The first is spam and general email mis-use. Essentially, this is what prompted the idea. I was angry/paranoid at the idea that some websites were selling their contact list to advertisers. Then there is the possibility that a website or friend's contact list is hijacked and an attacker now knows you and your meta-relation. The next problem is that email [sub-addressing](http://www.faqs.org/rfcs/rfc3598.html) does not work well. (There is also another problem with my implementation, we'll discuss this later.)

Solution: I thought of this problem a few years ago and tried to create a Firefox plugin that would handle centralized distribution of random email addresses. Centralizing this issue is not a solution. A plugin or addon is not a solution. Imagine talking to someone, yeah for real talking, and giving them an email address (let's call it a DYN address), you wont have easy access to such a plugin. Of course you could use a mobile app but at this point we're managing to many pieces of software (for me at least). I wanted the ability to generate a new email address for each application, service, signup card, conversation. Essentially a set of domain forwarders. I used EXIM for an MTA and forwarded mail from 20 subdomains to a small script. I've included a diagram to the right which shows email to a DYN for "dynamic email address" which is represented by a machine. The machine can interact with a database, if needed, then create another email with a modified reply-to header. The script can be configured to encapsulate the original email or forward it with small modifications. This new email arrives at a user's inbox and they see the sender as the original, but the recipient as themselves. However, the _reply-to_ has changed, this is to prevent the user from sending email from their real email address. Instead the _reply-to_ address will be directed back to the script which will forward the message as if it were sent from DYN. This proxies the user from the DYN email address.

Now there is a huge list of fancy options that email supports. However the script is only looking at the header information for the _from_ address, _subject_, _time_, and _to_ address. The _subject_ and _time_ are used to create an event. The _from_ address is stored as the originator of the event and the the _to_ address is stored as the receiver. The event is given an index and the _reply-to_ is modified to contain the DYN email address plus the event ID. When the email comes back through the script the event ID is stripped from the _to_ field and the event information is queried.

I mentioned checking a database when an email arrived at the mail proxy. I wanted to assure that only valid email addresses (one's that were used) were forwarded. Now this violates my ability to generate a dynamic address during a conversion only slightly. See, email addresses that are used to sign up for services are entered into the database. Additionally I maintain a short list of one-time-use addresses that I can hand out at will. If I want to convert one of these to a multi-use address I can subsequently add it. The database is also used to store descriptions of why the DYN address was used. For single person you'd most likely keep the list of valid addresses in memory instead of using a database.

Conclusion: Although this sounds a bit complicated, it has worked a few times for tracking down which websites post email addresses, and where spam comes from. And if I don't want to use a website or service any more, I can simply delete the address from my accepted list (no spam blocks or filters). It's sort of a firewall for email, with a default deny all. :) By the way, if you search for 'Proxied Email' you'll find Facebook's wiki page, seems like they had a similar [idea](http://wiki.developers.facebook.com/index.php/Proxied_Email). I'm sure others have too...

The only difficult part is achieving true anonymity if I'm the only one using the domains with dynamic addresses.

"Oh look at this person with email address: jimbo35@gophj.prosauce.org, and another dude777@llsfo.prosauce.org. Think they're the same person? ...Na"

But anonymity is not the point, if someone wants to find out my true identify go for it, I'm sure they're worth the unproxied conversation.
