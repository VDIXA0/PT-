# Nmap Host Discovery on Your Home Network (iphere/24)

Practice every host discovery technique from the TryHackMe room on a network you own.

**Only scan networks you own or have written permission to test.** Don't run this at a school, office, caf&eacute;, or on a neighbor's Wi-Fi.

Your machine: `iphere` &rarr; your network: `iphere/24` (256 addresses).

---

## 0. Setup

```bash
sudo apt update
sudo apt install -y nmap tcpdump masscan
```

Confirm your IP and interface name:

```bash
ip a
```

Look for `iphere`. The interface is probably `eth0` (wired) or `wlan0` (Wi-Fi). Use that name wherever this README says `IFACE`.

**If Kali is a VM:** set its network adapter to **Bridged**. On NAT you'll get a different range (like `10.0.2.x`) and ARP scans won't see your real devices.

Make a folder for your notes:

```bash
mkdir -p ~/homescan && cd ~/homescan
```

---

## 1. Target specification (no packets sent)

`-sL` lists targets without scanning. `-n` turns off DNS.

```bash
nmap -sL -n iphere/24            # whole subnet
nmap -sL -n iphere           # range: 11 addresses
nmap -sL -n iphere iphere   # range plus single host
nmap -sL -n iphere/30            # tiny subnet: 4 addresses
printf "iphere\niphere\n" > hosts.txt
nmap -sL -n -iL hosts.txt             # targets from a file
```

Questions:
1. How many addresses does `iphere` expand to?
2. How many are in `iphere/30`, and which?
3. How many addresses does `iphere` expand to?

---

## 2. ARP scan (same subnet only)

Terminal 2, watch the packets:

```bash
sudo tcpdump -ni IFACE arp
```

Terminal 1, scan:

```bash
sudo nmap -PR -sn -n iphere/24
```

Save the result as your **baseline**:

```bash
sudo nmap -PR -sn -n iphere/24 | tee baseline.txt
```

Questions:
- How many hosts are up? Which one is your router (usually `.1`)?
- In tcpdump, what is the destination MAC of each request? (Broadcast, `ff:ff:ff:ff:ff:ff`)
- Which IP is your own machine?

Run `-PE` without `--disable-arp-ping`:

```bash
sudo nmap -PE -sn -n iphere/24
```

Same result as ARP? Why? (On a local subnet, privileged Nmap uses ARP regardless.)

---

## 3. Simulate a remote network

ARP always wins on your own subnet, so disable it to test the other probes. Run these one at a time with `sudo`:

```bash
sudo nmap -sn -n --disable-arp-ping -PE  iphere/24 | tee pe.txt
sudo nmap -sn -n --disable-arp-ping -PP  iphere/24 | tee pp.txt
sudo nmap -sn -n --disable-arp-ping -PM  iphere/24 | tee pm.txt
sudo nmap -sn -n --disable-arp-ping -PS  iphere/24 | tee ps.txt
sudo nmap -sn -n --disable-arp-ping -PS80,443,8080 iphere/24 | tee ps_multi.txt
sudo nmap -sn -n --disable-arp-ping -PA  iphere/24 | tee pa.txt
sudo nmap -sn -n --disable-arp-ping -PU  iphere/24 | tee pu.txt
```

Meaning of each:

| Flag | Probe | Expected reply from a live host |
|---|---|---|
| `-PE` | ICMP echo (type 8) | ICMP echo reply (type 0) |
| `-PP` | ICMP timestamp (type 13) | ICMP timestamp reply (type 14) |
| `-PM` | ICMP address mask (type 17) | Address mask reply (type 18) |
| `-PS` | TCP SYN to port 80 | SYN/ACK (open) or RST (closed) |
| `-PA` | TCP ACK to port 80 | RST |
| `-PU` | UDP to a likely-closed port | ICMP port unreachable |

To watch the packets, run this in terminal 2 during each scan:

```bash
sudo tcpdump -ni IFACE 'not arp and host iphere'
```

---

## 4. Compare results (fill this in)

Put the hosts from your ARP baseline down the side. Mark ✔ if the probe found it.

| Device | IP | ARP | -PE | -PP | -PM | -PS | -PS80,443,8080 | -PA | -PU |
|---|---|---|---|---|---|---|---|---|---|
| Router | .1 | ✔ | | | | | | | |
| This PC | .16 | ✔ | | | | | | | |
| Phone | | ✔ | | | | | | | |
| TV / other | | ✔ | | | | | | | |

Questions:
1. Which device was found by ARP but missed by every other probe?
2. Which probe found the most devices? Which found the fewest?
3. Which devices answered `-PP` but not `-PE`?
4. Why does `-PM` find nothing?
5. Which devices were found by `-PU`? Why does UDP work against a host that has no UDP service?

Common patterns on home networks:
- Phones often ignore ICMP echo to save battery.
- Windows PCs drop echo by default (host firewall).
- Routers answer nearly everything.
- Linux machines usually answer echo and timestamp, but not address mask.

---

## 5. Build the best single command

```bash
sudo nmap -sn -n --disable-arp-ping -PE -PP -PS80,443 -PA80 -PU iphere/24 | tee best.txt
```

Questions:
- How many hosts does it find vs. the ARP baseline?
- Which hosts are still missing? What does that tell you about those devices' firewalls?
- Add more TCP ports and see if anything new appears: `-PS22,80,443,445,3389,8080`

---

## 6. Privileged vs. unprivileged

Run the same SYN ping with and without root:

```bash
sudo nmap -sn -n --disable-arp-ping -PS80 iphere/24
nmap -sn -n --disable-arp-ping -PS80 iphere/24
```

Why can it differ? With root, Nmap sends a raw SYN and never completes the handshake. Without root, Nmap uses the OS `connect()` call, which completes the 3-way handshake if the port is open. Use tcpdump to see the difference (SYN, SYN/ACK, ACK vs. SYN, SYN/ACK, RST).

---

## 7. Masscan (optional)

```bash
sudo masscan iphere/24 -p80,443 --rate 100
```

Keep the rate low (`--rate 100`). Masscan is very aggressive, and a high rate can overwhelm a home router. Compare its speed and results with Nmap's `-PS80,443` scan.

---

## 8. Discovery vs. skipping discovery

Pick an IP in your range that nothing uses, for example `iphere`:

```bash
time sudo nmap -n --disable-arp-ping -sn iphere
time sudo nmap -n --disable-arp-ping -Pn -p 1-1000 iphere
```

Why is `-Pn` slow? It treats every address as up and port-scans it, waiting on timeouts. Multiply that by 65,536 addresses in a /16 and it becomes impractical. That's why host discovery matters.

---

## 9. Write your report

For each device you found, write:

1. Its IP and what you think it is
2. Which probes it answered and which it ignored
3. The firewall behavior you can infer (e.g. "drops ICMP echo, allows ARP only")
4. The shortest probe that finds it

Final question: if you had only run plain `nmap iphere/24` as a remote attacker, which devices would your report be missing?

---

## Troubleshooting

**Only a few hosts show up / ARP scan finds nothing**
You're not on the same subnet. Check `ip a`. On a VM, switch to Bridged networking.

**Everything shows as up (256 hosts)**
Some routers or VPNs answer for every address (proxy ARP). Turn off any VPN and retry.

**"You requested a scan type which requires root privileges"**
Add `sudo`.

**Scans take very long**
Add `-T4`, or shrink the target range, e.g. `iphere`.

**Wrong interface for tcpdump**
List interfaces with `ip a`, and use that name in place of `IFACE`.

**Router warns or blocks you**
Some routers rate-limit scans. Wait a few minutes, lower speed with `-T2`, and keep masscan at a low rate.

---

## Quick answers

- Phase 1: 11 addresses; 4 addresses (.0, .1, .2, .3); 6 addresses.
- `-PM` finds nothing because address mask requests were deprecated (RFC 6918) and most systems don't answer them.
- `-PU` works because a closed UDP port makes the target send back an ICMP port unreachable, proving it is up.
- ARP always finds everything on the local subnet because firewalls rarely filter it, which is why it's useful in internal pentests and post-exploitation.
