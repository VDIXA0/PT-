# Passive Recon

Practical lab commands for passive reconnaissance on `example.com` (Kali Linux).
For the explanation of what passive recon is and how it works, see `Passive-Recon-Guide.pdf`.

## 1. Create the project

```bash
mkdir -p ~/Passive-Recon/{Report,Evidence,Screenshots}
cd ~/Passive-Recon
ls -R
```

## 2. Install and verify tools

```bash
sudo apt update
sudo apt install dnsutils whois -y
dig -v
whois --version
```

## 3. WHOIS

```bash
whois example.com
whois example.com > Evidence/whois.txt 2>&1
cat Evidence/whois.txt
```

## 4. DNS enumeration

```bash
dig example.com A       > Evidence/dns-a.txt
dig example.com AAAA    > Evidence/dns-aaaa.txt
dig example.com MX      > Evidence/dns-mx.txt
dig example.com NS      > Evidence/dns-ns.txt
dig example.com TXT     > Evidence/dns-txt.txt
dig example.com SOA     > Evidence/dns-soa.txt
dig example.com ANY
```

Short answer-only output:

```bash
dig example.com A +noall +answer
```

## 5. Extra checks

```bash
dig @1.1.1.1 example.com A
nslookup example.com
```

## 6. Verify evidence

```bash
ls -lh Evidence/

for file in Evidence/*.txt; do
    echo "========== $file =========="
    cat "$file"
    echo
done
```

## 7. Publish to GitHub

```bash
cd ~/Passive-Recon
git init
git add .
git commit -m "Add Passive Recon lab"
git branch -M main
git remote add origin https://github.com/<your-username>/Passive-Recon.git
git push -u origin main
```

> Authorized or practice targets only. Check `Evidence/whois.txt` for personal data before pushing.
