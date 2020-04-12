---
layout: post
title: 'HackTheBox: Swagshop Writeup'
published: true
---
## Summary ##
This box is a PHP-based online store, running on a content-management system (CMS) called Magento. Compromising this box required using quite a sneaky little vulnerability called [Froghopper](https://www.foregenix.com/blog/anatomy-of-a-magento-attack-froghopper). So without further ado, let's get into it!

## Reconnaisance Phase ##
As always with HTB targets, I ran an NMap scan, just to see what services could be found: `nmap -v -sS -A -Pn -T5 -p- 10.10.10.140`. Nothing interesting showed up, just HTTPD and SSH. Nothing worth poking at directly. 

Next up is a quick browse of the website itself. It's a fairly standard-looking eCommerce site, "selling" a few products and such. There's a login functionality, too, and we know that CMS products tend to have an admin interface built in to the site. Maybe we can see if we can compromise that, as a first objective.

## Obtaining CMS Admin Rights ##
A quick look around the exploit databases online reveals an RCE vulnerability in the Magento developer login system. An implementation of this vulnerability is available [on ExploitDB](https://www.exploit-db.com/exploits/37977), written in Python. The exploit is pretty simple; it's an SQL injection against one of the admin endpoints. Running this exploit creates an admin user with username `forme` and password `forme`. Now we can take a look around the admin interface. We want to achieve remote code execution to gain access to the server itself, ideally, so let's see if we can do that.

These sorts of CMS backends often have the ability to upload some form of code files, to allow developers to update the site's functionality on-the-fly. So we can look to see if we can take advantage of that. A little Googling about Magento tells us that it has a "package" system for this purpose. That sounds promising. Time to get our shell put together (in PHP, remember). I make quite a simple shell, not bothering with Metasploit unless it's necessary.

We go to Github, which has a handy repository [for a backdoor package](https://github.com/lavalamp-/LavaMagentoBD/tree/master/Backdoor%20Code) that we can use. We simply put in our own shell. Now, we just have to upload this to Magento. We go to the Magento Connect Management tool, and... 404 Not Found. Alas! As it turns out, due to the nature of this exploit (which essentially overwrites a given endpoint - /index by default), every time someone ran it without changing the endpoint which gets overwritten, the site became unusable for everyone else. So it seems that the owner of the box disabled the package-upload functionality. We've been thwarted. Back to the drawing board.

## Round 2: Froghopper ##
All is not lost, however. We have a backup plan: Froghopper. This is a more round-about way to achieve RCE, but more satisfying for it, in my opinion :) We start by grabbing some random JPEG image (in a real attack, we'd want to choose an image carefully so that it fits with the site's theme, for stealth purposes). We open that image up in something like Notepad or GEdit, and paste our PHP shell code at the end.

The next step is to upload that image as the thumbnail for a product _category_. It must be a category, rather than simply an individual product thumbnail, since it seems that validation is applied to product images and not category thumbnails. I have no idea why the implementations differ. Let that be a lesson to have consistency across your app when it comes to security. Now we have our PHP code in place, but it isn't executing yet. That's the next step.

How might we execute the code we've uploaded? Well, as it turns out, administrators can create email newsletter templates via the admin interface. These templates can point to code files, to allow the newsletters to be dynamic in various ways. Are you thinking what I'm thinking? Well, hold your horses for a moment. By default, Magento restricts template code to being read from a specific directory. This directory, funnily enough, is not where our image got uploaded to. So we have to take a quick detour to the config panel to "Enable Symlinks". _Now_ we can create our little newsletter. Go to the newsletter panel, and set up the newsletter to have a contentlike so:
```
{templateCode}
```
We then go to our terminal and run our NetCat listener: `nc -lvp 4444` (4444 is the port I chose for my shell to run on). We then preview the email template, and voilà, we're in!

## Flags ##
Now that we have our shell, let's look for the flags. The user flag is easy, we already have access to that. Unfortunately, however, we are not running as root (boooooo), so we need to escalate privileges in some way. So, let us do a little reconnaisance of what permissions we do have, and how we might escalate our privs. Running `sudo -l` shows that we can run Vi in the `/var/www/html` directory, as root, without needing to enter our password. Can we do something with this? Well, Vi has the handy feature of being able to run commands from within the application itself! So, let's `sudo /usr/bin/vi /var/www/html/foo.sh`. We can then run a command by entering `:!cat /root/root.txt`. Root flag == Pwnd!

## Conclusions ##
I enjoyed this box. It had a good balance of sneaky attacks and vulnerabilities, and CVE-style known vulnerabilities. It demonstrates some important aspects of security, around being consistent with how you apply your controls, and always always always validating user input!

Anyway, I hope you enjoyed this, and watch this space for the next write-up!
