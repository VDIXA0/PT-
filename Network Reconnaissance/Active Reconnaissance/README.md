# Network Utilities & Service Enumeration Lab — Command Reference

**Platform:** Kali Linux
**Tools:** Ping, Traceroute, Nmap, Telnet, Netcat, Curl
**Category:** Network Reconnaissance

> **Placeholders used in this file**
>
> | Placeholder | Meaning |
> |---|---|
> | `<TARGET_IP>` | IP address of the target machine |
> | `<HTTP_PORT>` | Port of the HTTP service on the target |
> | `<FTP_PORT>` | Port of the FTP control service on the target |
> | `<LISTENER_PORT>` | Any free local port used for the Netcat listener exercise |

---

## 1. Lab Environment Preparation

```bash
mkdir -p ~/Network-Utilities-Lab/{scans,evidence,reports}
cd ~/Network-Utilities-Lab
pwd
ls -R
```

---

## 2. Ping — ICMP Connectivity Test

```bash
ping -c 5 <TARGET_IP>
```

- `-c 5`: send five ICMP Echo Requests.
- Key fields: `icmp_seq`, `ttl`, `time`, `packet loss`.
- A failed ping does not always mean the host is offline; ICMP may be blocked.

---

## 3. Traceroute — Network Path Discovery

Installation:

```bash
sudo apt update
sudo apt install traceroute -y
```

Default (UDP) traceroute:

```bash
traceroute <TARGET_IP>
```

TCP traceroute:

```bash
sudo traceroute -T -p <HTTP_PORT> <TARGET_IP>
```

- `-T`: use TCP packets instead of UDP.
- `-p`: destination port.
- `*` means no response within the timeout.

---

## 4. Nmap — Service and Port Detection

Service version detection:

```bash
nmap -sV -p <HTTP_PORT> <TARGET_IP>
```

Follow-up SYN scan with reason:

```bash
sudo nmap -Pn -sS -p <HTTP_PORT> --reason <TARGET_IP>
```

- `-sV`: detect service versions.
- `-Pn`: skip host discovery, treat host as online.
- `-sS`: TCP SYN scan.
- `--reason`: show why a port is in a given state.

Port states: `open`, `closed`, `filtered`.

Observed result (values hidden):

```text
<HTTP_PORT>/tcp   filtered   http
```

The scan could not determine whether the HTTP port was open or closed (possible firewall filtering).

---

## 5. Telnet — HTTP Banner Grabbing

```bash
telnet <TARGET_IP> <HTTP_PORT>
```

Then type the request followed by a blank line:

```http
GET / HTTP/1.1
Host: <TARGET_IP>
```

Exit: press `Ctrl + ]`, then type `quit`.

---

## 6. Netcat — HTTP Banner Grabbing

```bash
nc -nv <TARGET_IP> <HTTP_PORT>
```

Then type the request followed by a blank line:

```http
GET / HTTP/1.1
Host: <TARGET_IP>
Connection: close
```

- `-n`: no DNS resolution. `-v`: verbose.
- A TCP connection alone does not guarantee an HTTP response; a valid request is needed.

---

## 7. FTP Banner Grabbing

```bash
nc -nv <TARGET_IP> <FTP_PORT>
```

FTP servers normally send a welcome banner automatically on connect.

---

## 8. Netcat Listener and Client (Local)

Terminal 1 — listener:

```bash
nc -vnlp <LISTENER_PORT>
```

Terminal 2 — client:

```bash
nc -nv 127.0.0.1 <LISTENER_PORT>
```

Type a test message in either terminal and confirm it appears in the other.
This is plain TCP text communication, not a reverse shell.

---

## 9. Curl — HTTP Request Testing

```bash
curl -i http://<TARGET_IP>:<HTTP_PORT>/
```

- `-i`: include response headers in the output.
- Common status codes: `200`, `301`, `302`, `403`, `404`, `500`.

---

## 10. Saving Evidence

```bash
nmap -sV -p <HTTP_PORT> <TARGET_IP> -oN scans/services.txt
ping -c 5 <TARGET_IP> > evidence/ping.txt
traceroute <TARGET_IP> > evidence/traceroute.txt
```

Suggested layout:

```text
Network-Utilities-Lab/
├── scans/      services.txt
├── evidence/   ping.txt, traceroute.txt
└── reports/    lab-info.txt, netcat-listener.txt
```
