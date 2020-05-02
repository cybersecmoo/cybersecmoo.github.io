---
layout: post
title: 'HackTheBox: OpenAdmin Writeup'
published: true
---
## Summary ##
This is an Easy-rated Linux box, using a CVE, and some basic source code auditing/modification.

## Reconnaisance Phase ##
We start by using nmap as usual, running both a quick script-scan: `nmap -sS -sC -Pn 10.10.10.171`, and a more complete (though without scripts) scan: `nmap -sS -Pn -p- 10.10.10.171`. We find only the SSH and HTTP ports exposed. So nothing particularly helpful there.

Next, we do some web crawling using DirBuster. Using the "medium" wordlist, we find what looks to be an interesting administrative console. Browsing to this page manually, we find that it is running some software called "OpenNetAdmin", on what appears to be an outdated version: 18.1.1. Outdated version means possible CVEs, so let us search exploit-db etc. for CVEs. We find one [here.](https://www.exploit-db.com/exploits/47691). It looks like some form of AJAX OS command injection exploit.

## Initial Foothold and First PrivEsc ##
With this exploit, we now have a foothold on the server. We are constrained to a low-privilege user, but we can at least access application configuration files. Searching around these files, we find some database access credentials, as well as a human user account name (`jimmy`). Let's try these credentials in various parts of the stack...

Hey! Turns out we can use them to log in to the box via SSH. This implies that `jimmy` duplicated his credentials over to the database system account (a big no-no!).

## Second PrivEsc ##
We then do some more local reconnaisance, and find the source code of the live application, as well as (more interestingly) the source code to some form of internal or development site. By looking around at the configuration of the internal site, we find that, once started, it changes its user ID to that of a second developer, `joanna`. This looks like a good target for our next privesc. So, we modify the code for the internal site such that it prints the contents of `joanna`'s private SSH key.

We then run the application, and make a `curl` request on localhost to retrieve the SSH key. Exfiltrating this to `ssh2john` --> `JohnTheRipper`, we chuck `rockyou.txt` at it, and quickly crack the passphrase.

## Final Privesc ##
We now have access to the box as `joanna`, via SSH. A quick `sudo -l` reveals that we can run `nano` as root, which means we have read access to all the files, including `root.txt`. Pwnd.

## Conclusions ##
This was a nice easy box, with a few different privescs to follow. Like a few of the other easy boxes, the final escalation requires abuse of passwordless sudo permissions.

