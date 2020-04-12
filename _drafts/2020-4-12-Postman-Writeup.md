---
layout: post
title: 'HackTheBox: Postman Writeup'
published: true
---

## Summary ##
This box is an interesting beginner one. It involves a lot of enumeration, and a little bit of perseverance. The box involves exploiting two services, primarily: Redis and Webmin. It also requires some offline password-cracking!

## External Reconnaisance Phase ## 

As usual with HTB targets, we run an NMap scan, to see what services are running: `nmap -v -sS -A -Pn -T5 -p- 10.10.10.160`. This reveals: SSH, as normal; HTTPD, which is running the simple webpage; 10000, which is the Webmin administration interface (interesting - we'll take note of that) and finally 6379, which is Redis (another one to note).
        
Let's take a closer look at this Webmin service. As an administration interface, it could be our golden ticket. A quick Google for CVEs reveals a couple of RCE-type vulnerabilities. The first requires some existing level of access, so that's one to keep in mind for later, maybe. The other doesn't require authentication, and also has a Metasploit module associated with it! This sounds promising. Whenever I am about to run an exploit, I always try to check the code and read around the exploit to see how it works. Whilst doing so, I found [a script](https://packetstormsecurity.com/files/154141/Webmin-Remote-Comman-Execution.html) which checks to see if the server is vulnerable to the exploit. Running the script reveals that the server is not vulnerable. Alas.
        
So let's inspect Redis instead. I wasn't familiar with it, so I took some time to read up about it, what the commands are and so on. Basically, it's a key-value store. You can install a CLI tool to run commands against the server. Looking on the CVE lists tells us that there is a vulnerability, with a Metasploit module associated. The vuln revolves around the fact that you can compile your own extensions for Redis (in C). These can then be copied out around the Redis cluster. So with an unsecured instance, you can spin up a rogue node, slave the victim to it, then copy the malicious module out to the victim. Running this exploit doesn't seem to work, though. The command which should have been registered is not usable. Hm. Let's try some other options before going too deep down the rabbit hole of trying to fix that one.

## Initial Foothold ##

Whilst investigating the CVEs, I found another one [based around SSH](https://packetstormsecurity.com/files/134200/Redis-Remote-Command-Execution.html). When you save out Redis keys and their values, they are written to a file specified by the "dbfilename" key. So what we do is we set the value of some key to an SSH private key that we create for this purpose: `cat private_key_rsa.txt | redis-cli -h 10.10.10.160 -x set exploit`. We then set the working directory of the Redis server to `~/.ssh`, and set the dbfilename to `authorized_keys`. By saving the config, we then get the Redis server to write a binary dump of the config to the authorized_keys file. This binary dump includes our SSH key, which should survive the dump intact. When we then try to log in via SSH, the SSH service finds a bunch of rubbish in the authorized_keys, as well as a valid entry. It then uses the valid entry (obviously :|). So we run this attack, and get ourselves our initial shell!

## Privilege Escalation ##

So let us enumerate a bit. We're quite restricted in what we can do. Without an actual login password, we can't run `sudo` so we can't check for a list of what commands we can run as root. We can try running things like Vi, Bash, and so on as root to see if any have no-login enabled. Unfortunately, no luck there. Next job is to start searching for interesting files. We look at the passwd file, and find that there is a user Matt. We look for SSH keys by running `find / -iname 'id_rsa*'`, which reveals something interesting: a backup SSH key, in the `/opt` directory. We write this out to our Kali machine, and see if we can crack it. To do this, we firstly run the ssh2john tool to convert the private key to JohnTheRipper format. Then we hit it with John, using the RockYou wordlist. This gives us the passphrase. Nice.
        
Let's now use that private key to log in to the server as Matt. Annnnd... Nothing. This is odd. We seem to connect properly, but then we get dropped immediately before getting a shell. A quick look at the sshd_config on the server (using our Redis user) shows that Matt is denied login via SSH (a kinda CTF-y constraint). So how do we get access? Let's take a few steps back. Maybe this password is used by Matt elsewhere? Maybe in the Webmin interface? Let's give it a go. Sure enough, Matt:computer2008 gives us access to Webmin! Wait a minute, this means we can run [that CVE](https://www.cvedetails.com/cve/CVE-2019-12840) we turned up way back at the beginning! Into Metasploit we go! Run it, and we get root. We can then access both the root and the user flags.

## Conclusions ##

This was an interesting one, and demonstrates the importance of securely configuring your database (and, indeed, other) services. There's a reason why broken configuration is number six on the [OWASP Top Ten](https://www.owasp.org/images/7/72/OWASP_Top_10-2017_%28en%29.pdf.pdf). It also emphasises the importance of thorough enumeration when pentesting.
