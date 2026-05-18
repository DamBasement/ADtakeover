# 🎯 Active Directory Assessment Playbook & Verification Checklist
> **Metodologia Operativa Sequenziale: Dai Leak Esterni al Foothold Autenticato**
> Questo documento serve come traccia d'ingaggio e foglio di verifica per verificare lo stato di avanzamento e la correttezza dei passaggi eseguiti.

---

## 🗺️ Flusso Logico dell'Attacco
Il successo di un internal assessment dipende dal rigore metodologico. Non saltare i passaggi passivi prima di lanciare tool rumorosi.
1. **External Recon & OSINT**: Caccia ai leak storici e segreti esposti.
2. **Passive Network Analysis**: Ascolto del traffico L2 (Black Box).
3. **Active Host & Service Discovery**: Mappatura del perimetro interno.
4. **Name Resolution Poisoning**: Avvelenamento dei protocolli broadcast (LLMNR/NBT-NS).
5. **Unauthenticated AD Enumeration**: Sfruttamento di sessioni nulle o anonime (SMB/LDAP/Kerberos).
6. **Foothold & Authenticated Post-Enumeration**: Consolidamento e mappatura con credenziali valide.

---

## 🛠️ checklist Operativa e Sequenziale

### 📦 1. External Recon & OSINT
*Fase preliminare per identificare informazioni sull'infrastruttura e credenziali compromesse prima ancora di interfacciarsi con la rete interna.*
- [ ] **Mappatura dei blocchi IP e ASN**
  - Tool: [BGP Toolkit](https://bgp.he.net/)
  - Obiettivo: Ricercare i blocchi di indirizzi assegnati all'organizzazione e identificare l'ASN di riferimento.
- [ ] **Validazione dello Scope di Rete**
  - Tool: [ViewDNS.info](https://viewdns.info/)
  - Obiettivo: Convalidare il perimetro d'azione e scoprire host esterni raggiungibili non dichiarati esplicitamente dal cliente.
- [ ] **Scansione dei Repository Pubblici**
  - Tool: [TruffleHog](https://github.com/trufflesecurity/truffleHog)
  - Obiettivo: Automatizzare la ricerca di segreti, chiavi API o credenziali hardcoded lasciate erroneamente nei push del codice (es. GitHub).
- [ ] **Analisi di Data Breach Storici**
  - Tool: [DeHashed API Script](https://github.com/mrb3n813/Pentest-stuff/blob/master/dehashed.py)
  - Comando:
    ```bash
    sudo python3 dehashed.py -q inlanefreight.local -p
    ```
  - Obiettivo: Estrarre password in chiaro o hash associati al dominio del cliente da violazioni pubbliche passate.

---

### 🔍 2. Initial Internal Enumeration (Host & Service Discovery)
*Una volta posizionato l'host di attacco all'interno del perimetro locale, mappare passivamente e attivamente la superficie d'attacco.*

#### 2.1 Analisi Passiva del Traffico (Fase iniziale / Black Box)
- [ ] **Intercettazione del Traffico di Livello 2 (L2)**
  - Tool: `tcpdump`, `Wireshark`, `tshark`
  - Obiettivo: Catturare pacchetti base (ARP, mDNS, LLMNR) per comprendere la topologia di rete senza generare rumore o alert.
- [ ] **Estrazione Automatica di Credenziali in Transito**
  - Tool: [Net-creds](https://github.com/DanMcInerney/net-creds)
  - Obiettivo: Analizzare costantemente l'interfaccia di rete o un file `.pcap` alla ricerca di credenziali e hash scambiati in chiaro (es. FTP, HTTP, SMB).

#### 2.2 Scansione Attiva di Rete
- [ ] **Host Discovery Massivo via ICMP**
  - Tool: `fping`
  - Comando:
    ```bash
    fping -a -s -g 172.16.5.0/24 -q
    ```
  - *Note operative sui flag:* `-a` (mostra solo i target attivi), `-s` (stampa statistiche finali), `-g` (genera il range CIDR), `-q` (silenzia i singoli timeout).
- [ ] **Scansione dei Servizi AD Critici (Se non richiesto approccio stealth)**
  - Tool: `Nmap`
  - Comando:
    ```bash
    sudo nmap -v -A -iL hosts.txt -oN scansione_interna.txt
    ```
  - Obiettivo principale: Identificare la presenza di Domain Controller (DC), File Server, SQL Server, Exchange, Web Server e mappare i servizi core di AD:
    * **Kerberos** (Porta 88)
    * **DNS** (Porta 53)
    * **SMB** (Porta 445)
    * **LDAP** (Porte 389 / 3268)
    * **NetBIOS** (Porte 137-139)

---

### ☣️ 3. Name Resolution Poisoning (Man-in-the-Middle)
*Sfruttare i meccanismi di fallback di Windows per intercettare l'autenticazione di rete.*

- [ ] **Avvelenamento Broadcast da Macchina di Attacco Linux**
  - Tool: **Responder**
  - Obiettivo: Rispondere in modo malevolo alle richieste broadcast fallite di LLMNR, NBT-NS e mDNS per forzare i client a inviare i loro hash di rete NTLMv1/NTLMv2.
- [ ] **Avvelenamento Broadcast da Macchina di Attacco Windows**
  - Tool: **Inveigh**
  - Obiettivo: Eseguire la medesima attività di Responder qualora l'assessment richieda l'uso esclusivo di una workstation Windows aziendale o un host già compromesso (intercetta IPv4/IPv6, LLMNR, DNS, mDNS, NBNS, DHCPv6, SMB, HTTP, LDAP).
- [ ] **Cracking degli Hash Offline**
  - Tool: `Hashcat` / `John the Ripper`
  - Obiettivo: Convertire gli hash catturati in password in chiaro.
  - Target prioritario: Ottenere un foothold iniziale o espandere i privilegi se l'hash catturato appartiene a un account amministrativo.

---

### 👥 4. Unauthenticated AD User Enumeration
*Se l'avvelenamento non produce frutti, l'obiettivo è compilare una lista pulita di utenti reali del dominio per preparare attacchi di Password Spraying.*

- [ ] **Verifica di Sessioni SMB NULL (Senza Autenticazione)**
  - Sfrutta configurazioni legacy o Domain Controller aggiornati mantenendo permessi deboli.
  - *Estrazione via enum4linux-ng:*
    ```bash
    enum4linux -U 172.16.5.5 | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"
    ```
  - *Verifica interattiva via rpcclient:*
    ```bash
    rpcclient -U "" -N 172.16.5.5
    # Comandi interni consigliati: querydominfo, enumdomusers
    ```
  - *Verifica rapida via CrackMapExec / NetExec:*
    ```bash
    crackmapexec smb 172.16.5.5 --users
    ```
- [ ] **Verifica di Accessi LDAP Anonimi (LDAP Anonymous Bind)**
  - *Estrazione manuale via ldapsearch (con filtro utenti dedicato):*
    ```bash
    ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "
    ```
  - *Estrazione automatizzata via Windapsearch:*
    ```bash
    ./windapsearch.py --dc-ip 172.16.5.5 -u "" -U
    ```
- [ ] **Enumerazione Stealth via Pre-Autenticazione Kerberos**
  - Da usare se SMB e LDAP anonimi sono opportunamente disabilitati. È un attacco silenzioso perché i fallimenti non generano l'evento Windows ID 4625 (fallito accesso).
  - Tool: [Kerbrute](https://github.com/ropnop/kerbrute.git) *(richiede compilazione)*
  - Wordlist consigliata: [statistically-likely-usernames](https://github.com/insidetrust/statistically-likely-usernames) o generate via OSINT tramite `linkedin2username`.
  - Comando:
    ```bash
    kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.5.5 /opt/jsmith.txt -o valid_ad_users
    ```

---

### 🔑 5. Password Policy Analysis & Foothold Consolidato

- [ ] **Estrazione dei Criteri Password (Password Policy)**
  - ⚠️ **MANDATORIO prima di qualsiasi attacco di Password Spraying** per evitare il blocco massivo degli account aziendali (*Account Lockout*).
  - *Senza credenziali:* Sfruttare la sessione SMB NULL tramite `rpcclient` (comando `querydominfo`).
  - *Con credenziali valide (recuperate da leak/OSINT):*
    ```bash
    crackmapexec smb 172.16.5.5 -u <USER> -p <PASSWORD> --pass-pol
    ```
  - *Da host Windows locale già compromesso:* `net accounts`
  - Elemento da validare: **Minimum password length** e **Account lockout threshold**.
- [ ] **Estrazione Massiva di Informazioni post-Foothold**
  - Una volta ottenuta una credenziale valida funzionante (es. `htb-student`), dumpare legalmente l'intera lista utenti e computer del dominio.
  - Comando:
    ```bash
    sudo crackmapexec smb 172.16.5.5 -u htb-student -p Academy_student_AD! --users
    ```
- [ ] **Analisi dei Percorsi di Attacco con BloodHound**
  - Collezionare i dati di dominio e avviare l'ingestione grafica per mappare le relazioni di trust, ACL deboli e percorsi di privilege escalation verso il Domain Admin.

---

## 🧐 Come capire se il tuo target / documento è OK?
Controlla l'output dei tuoi test a fronte di questi criteri di qualità:
1. **La lista utenti è "pulita"?** I comandi come `cut` e `grep` inseriti sopra servono a lasciarti un file con *un utente per riga*, senza log o caratteri speciali. Se vedi scritte come `user:[admin]`, pulisci il file o i tool di password spraying falliranno.
2. **Hai verificato il blocco account?** Se lanci un attacco spray senza aver estratto il valore `--pass-pol`, stai rischiando di bloccare l'intera azienda. Se la policy dice "Lockout Threshold: 5", tu puoi testare al massimo **1 password ogni 30 minuti** su tutta la lista utenti.
3. **Il formato dei percorsi LDAP è corretto?** Ricorda che per i filtri LDAP il dominio va sempre spezzato (es. `inlanefreight.local` diventa obbligatoriamente `DC=INLANEFREIGHT,DC=LOCAL`).
AD_Pentesting_Playbook.md
Visualizzazione di AD_Pentesting_Playbook.md.
