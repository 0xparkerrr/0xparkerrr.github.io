---
layout: post
title: CTF Writeup - Mr Robot
description: A writeup covering the Mr. Robot Vulnhub machine
summary: A writeup covering the Mr. Robot Vulnhub machine
tags: writeups ctf priv-esc wordpress
minute: 10
---

Table of Contents
- [Overview](#overview)
- [Technical Analysis](#technical-analysis)
    - [Root Webpage](#root-webpage)
    - [WordPress](#wordpress)
        - [Enumerating Usernames](#enumerating-usernames)
        - [Uploading Reverse Shell](#uploading-reverse-shell)
    - [Post Exploitation](#post-exploitation)
    - [Privilege Escalation](#privilege-escalation)
# Overview
In this blogpost, we'll be covering how I solved the Mr. Robot from Vulnhub. There are three flags hidden, and each key is progressively difficult to find.

I ran a Kali box alongside the machine.

# Technical Analysis
Before we get started, we need to know what the machine's address is. Since we are on the same internal network, we can scan the range. My range happens to be 192.168.56.0/24.
![](/assets/analysis/mr-robot/nmap.PNG)
>You can also run netdiscover, but I chose nmap.

We get a response saying that the machine 192.168.56.107 has a closed SSH port and two web services running. Let's take a closer look at the web service.

## Root Webpage
The root webpage is an interactive terminal session that has easter eggs from the TV show, *Mr. Robot*, which is pretty cool, but not what we're looking for.

The next step would be to fuzz for directories. We'll go with a simple scan with `ffuf`, and another one recursively if needed.
![](/assets/analysis/mr-robot/ffuf.PNG)

The output of the scan was a bit wonky, but we do find that there's a directory at `/blog`, `/site` and a status 302 at `/wp-admin` indicating there's a WordPress blog being hosted. When we visit `/site`, there's an incomplete WordPress blog:
![](/assets/analysis/mr-robot/site.PNG)

## WordPress
When it comes to exploiting WordPress sites, I first check to see if directory indexing is possible. This is because if it returns a 200 response, we should be able to upload a reverse shell if we get the right credentials.

If we look at the HTML source code, it tells us the theme that the site is using and where it's located:
{% highlight html %}
<!--[if lt IE 9]>
	<script src="http://192.168.56.108/wp-content/themes/twentyfifteen/js/html5.js"></script>
	<![endif]-->
{% endhighlight %}

If we request just `/wp-content`, we see a blank page which means the request was successful. This means directory indexing is possible.
![](/assets/analysis/mr-robot/wp-content.PNG)

--- 

Since the WordPress site is incomplete, I can't really enumerate for any users manually. What I'll do is use `WPScan` to see if it'll get me some good info. I'll run a regular scan and another scan that will enumerate for users.
{% highlight shell %}
┌──(kali㉿kali)-[~/Documents/mr_robot]
└─$ wpscan --url http://192.168.56.108/

┌──(kali㉿kali)-[~/Documents/mr_robot]
└─$ wpscan --enumerate u --url http://192.168.56.108/
{% endhighlight %}

Unfortunately, it did not find any users for us. However, in the scans, it does tell us that `robots.txt` is hidden, which holds our first key:
![](/assets/analysis/mr-robot/robots.PNG)

It also holds another file called `fsocity.dic`. When we open it with a text editor, we see it's a dictionary of words ; Words that we can use to enumerate valid usernames.

### Enumerating Usernames
![](/assets/analysis/mr-robot/wp-login.PNG)

Most people don't know, but WordPress has a known weakness that tells you if a user exists. If we enter invalid username and passwords, we get a message saying, `ERROR: Invalid username`. Basically, WordPress doesn't have a one generic error message for all scenarios. 

Knowing this, we can use `hydra` to bruteforce the login page only that we are not expecting a valid login request but an error message that *isn't* `ERROR: Invalid username`.

To craft properly execute the tool, we need to make sure we include all the POST data that is submitted when we try to log in. 
![](/assets/analysis/mr-robot/post.PNG)
![](/assets/analysis/mr-robot/hydra.PNG)
>We get a hit!

With the dictionary provided, we get a hit for the user `Elliot`. When we try to login with the wrong password, we get a different error message:
![](/assets/analysis/mr-robot/error-pass.PNG)

This means we can do the same thing again with hydra. However, our dictionary as is will not work. I've tried multiple variations:
- combining credentials
- generating alternative combolist

The only one that did seem to work is by piping the dictionary against `sort` and `uniq`. This makes it so that it goes from numerics first and then letters and characters. Now we just need to slightly modify our previous command:
{% highlight shell %}
┌──(kali㉿kali)-[~/Documents/mr_robot]
└─$ hydra -l elliot -P fsocity.txt 192.168.56.108 -s 80 http-post-form '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&testcookie=1:F=The password you entered'
{% endhighlight %}

And we get a successful hit!

![](/assets/analysis/mr-robot/hit.PNG)

### Uploading Reverse Shell
![](/assets/analysis/mr-robot/dashboard.PNG)
![](/assets/analysis/mr-robot/users.PNG)

Since the user we logged in as is an admin, we can look to upload a reverse shell. Recall back to earlier when we found that directory indexing is possible. We'll take advantage of a theme that is not in use and modify an existing PHP file.

First we'll head over to Appearance > Editor. This is the theme editor. We will upload our reverse shell to the `404.php` file. Any shell works, but here I just used the one from pentestmonkey.
![](/assets/analysis/mr-robot/themes.PNG)
![](/assets/analysis/mr-robot/upload.PNG)

All that's left is to create a listener on our end and make a request to the `404.php` file. 
![](/assets/analysis/mr-robot/shell.PNG)

## Post Exploitation
Now that we've gained a reverse shell to the system, we can upgrade our shell with `python -c 'import pty;pty.spawn("/bin/bash")'`.
>This will only work if the machine has Python installed. You would just need to run `which python` to see if it is.

Taking a look around, we find there's a directory at `/home/robot/` with two files. We don't have the right permissions to read the text file. We can, however, read the `password.raw-md5` file which looks like the credentials for the `robot` user. 
![](/assets/analysis/mr-robot/shell.PNG)

There are a couple of ways we can crack the password. Because MD5 isn't really secure, if the original plaintext is also insecure we can use tools like [Crackstation](https://crackstation.net/) or [Decodify](https://github.com/s0md3v/Decodify). I've taken a liking to Decodify recently, so that's what I use, but it's always good to have multiple options.

![](/assets/analysis/mr-robot/decodify.PNG)
![](/assets/analysis/mr-robot/crackstation.PNG)
*Crackstation: very useful for accuracy*

Now that we've cracked the MD5 hash, we can now login as `robot` and read the flag.
![](/assets/analysis/mr-robot/su.PNG)

## Privilege Escalation
Up until this point, we've gained an initial foothold into the machine by exploiting a WordPress site and performed lateral movement by accessing a low-privileged account. Our goal now is to gain root privileges.

We can check to see if this user can run any commands with sudo privileges with `sudo -l` but this is what we get:

![](/assets/analysis/mr-robot/sudo-l.PNG)

Unfortunately, that didn't work. But there is another way for users to run files even if we don't have sudo permissions: SUID bits. If this permission is set on a file, then an alternate user can execute that file as if they were the owner. A command that will help with this is `find / -perm /4000 -type f 2>/tmp/2`. This will list all files that have the SUID bit set.

![](/assets/analysis/mr-robot/find-perm.PNG)

And indeed we do get a good number of files, but most of them seem to be ordinary files. The binary at `/usr/local/bin/nmap` does seem interesting, so let's see what we can do with it.

---

![](/assets/analysis/mr-robot/nmap2.PNG)

The version number of this binary is 3.81. If we look around in [GTFOBins](https://gtfobins.github.io/gtfobins/nmap/), it tells us our version of nmap lies in the range of the interactive mode which allows us to break out of the current environment. 

When we try it, it works! All that's left is to get the last flag in the `/root/` directory.
![](/assets/analysis/mr-robot/root.PNG)
![](/assets/analysis/mr-robot/flag3.PNG)




