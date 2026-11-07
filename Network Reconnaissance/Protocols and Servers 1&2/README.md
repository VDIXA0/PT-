# Linux Networking Lab: All Commands

Command reference for practising network protocols in Kali Linux using the Docker lab.
Ports used by the lab: HTTP `8080`, FTP `2121`, SSH `2222`, Telnet demo `2323`.

---

## 1. Start Your Lab

```bash
cd linux_protocols_lab
docker compose up -d
docker compose ps
docker compose logs
```

Stop the lab:

```bash
docker compose down
```

Restart it:

```bash
docker compose restart
```

---

## 2. Basic Linux Networking Commands

```bash
ip addr
ip route
hostname
hostname -I
ifconfig
ss -tulnp
ss -lnt
netstat -tulnp
ping 127.0.0.1
```

Install useful tools:

```bash
sudo apt update
sudo apt install -y curl wget netcat-openbsd telnet ftp nmap openssh-client
```

---

## 3. Nmap: Service Discovery

```bash
nmap 127.0.0.1
nmap -sT 127.0.0.1
nmap -sV -p 8080,2121,2222 127.0.0.1
nmap -p 8080 127.0.0.1
nmap -p 2121 127.0.0.1
nmap -p 2222 127.0.0.1
```

---

## 4. HTTP: Port 80 / 8080

```bash
curl http://127.0.0.1:8080
curl -I http://127.0.0.1:8080
curl -v http://127.0.0.1:8080
curl http://127.0.0.1:8080/flag.html
```

Download a page:

```bash
wget http://127.0.0.1:8080/index.html
```

GET request using Netcat:

```bash
printf 'GET / HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n' | nc 127.0.0.1 8080
```

HEAD request using Netcat:

```bash
printf 'HEAD / HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n' | nc 127.0.0.1 8080
```

---

## 5. FTP: Port 21 / 2121

Connect:

```bash
ftp 127.0.0.1 2121
```

Credentials:

```
Username: student
Password: LabOnly_ChangeMe_123
```

Commands inside the FTP prompt:

```
help
pwd
ls
dir
status
ascii
binary
get README.txt
get notes.txt
bye
```

Additional FTP commands:

```
cd /
mkdir practice
delete notes.txt
close
quit
```

> Some commands depend on server permissions; the lab account may not be allowed to perform every operation.

---

## 6. SSH: Port 22 / 2222

Connect:

```bash
ssh -p 2222 student@127.0.0.1
```

Inside the SSH session:

```bash
whoami
hostname
pwd
ls
ls -la
ip addr
exit
```

Check the SSH port:

```bash
ss -tlnp
```

---

## 7. Telnet: Port 23

Terminal 1 (listener):

```bash
nc -lv 127.0.0.1 2323
```

Terminal 2 (client):

```bash
telnet 127.0.0.1 2323
```

Type a message in Terminal 2 and observe it in Terminal 1.

Exit Telnet:

```
Ctrl + ]
quit
```

> This demonstrates plain TCP communication, not a full Telnet server.

---

## 8. SMTP: Port 25

Check the default SMTP port locally:

```bash
ss -tulnp | grep ':25'
```

SMTP protocol commands:

```
EHLO localhost
HELO localhost
MAIL FROM:<student@localhost>
RCPT TO:<receiver@localhost>
DATA
QUIT
```

> These are protocol commands, not Linux shell commands. They require an SMTP server to process them.

---

## 9. POP3: Port 110

```
USER student
PASS example
STAT
LIST
RETR 1
QUIT
```

---

## 10. IMAP: Port 143

```
a1 LOGIN student example
a2 LIST "" "*"
a3 SELECT INBOX
a4 STATUS INBOX (MESSAGES)
a5 LOGOUT
```

> The mail commands (SMTP, POP3, IMAP) are for understanding the protocols; the Docker lab does not run mail servers.

---

## 11. HTTPS: Port 443

TLS information:

```bash
openssl s_client -connect example.com:443 -servername example.com
```

Certificate information:

```bash
curl -Iv https://example.com
```

Port reminders:

| Protocol | Port |
| -------- | ---- |
| HTTP     | 80   |
| HTTPS    | 443  |
| SSH      | 22   |
| Telnet   | 23   |

---

## 12. Apache and Nginx

Apache:

```bash
apache2 -v
systemctl status apache2
```

Nginx:

```bash
nginx -v
systemctl status nginx
```

The Docker lab uses Nginx. Check its logs:

```bash
docker logs protocol-lab-web
```

---

## 13. FTP Active and Passive Modes

Inside FTP:

```
passive
status
ascii
binary
```

| Mode    | Meaning                              |
| ------- | ------------------------------------ |
| Active  | Server initiates the data connection |
| Passive | Client initiates the data connection |
| ASCII   | Text-file transfer mode              |
| Binary  | Binary-file transfer mode            |

---

## 14. Final Practice Checklist

- [ ] Identify your IP using `ip addr`
- [ ] Identify listening ports using `ss -tulnp`
- [ ] Scan localhost using Nmap
- [ ] Connect to HTTP using `curl`
- [ ] Send HTTP GET using Netcat
- [ ] Download a page using `wget`
- [ ] Connect to FTP
- [ ] List FTP files using `ls`
- [ ] Download an FTP file
- [ ] Practise ASCII and Binary modes
- [ ] Connect using SSH
- [ ] Practise plain TCP using Netcat and Telnet
- [ ] Understand SMTP commands
- [ ] Understand POP3 commands
- [ ] Understand IMAP commands
- [ ] Inspect HTTPS certificates
- [ ] Identify Apache and Nginx

**Tip:** Repeat this lab until you can perform the commands without notes and explain what happens between the client and server.
